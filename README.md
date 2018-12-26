# ubuntu-yubikey-setup

This is a guide for configuring an Ubuntu 18.04-based distribution (such as elementaryOS 5.0 "Juno") with:
* Full disk Yubikey-backed encryption (using challenge-response mode)
* System wide Yubikey login (graphical lightdm + non-graphical TTYs)
* Yubikey challenge-response mode for SUDO
* Yubikey for SSH authentication

## Prerequisites
- An existing installation of an Ubuntu 18.04-based distro with full-disk encryption
- A 2-pack of Yubikeys (version 5 NFC), if you only have one Yubikey you can skip the steps for the second key.

### Install required packages
```
sudo apt install software-properties-common  # provides add-apt-repository
sudo add-apt-repository ppa:yubico/stable && sudo apt-get update
sudo apt install libpam-yubico yubikey-manager
```

### For each yubikey:
If you require that the Yubikey must be touched for each challenge-response operation, pass `--touch` to `ykman otp chalresp`. This greatly enhances the security of the challenge-response mode since it needs a physical confirmation.
```
ykman otp chalresp --touch --generate 2
ykpamcfg -2
```
If you on the other hand want the key to automatically send a response to each challenge, omit the `--touch`. **Note that this is less secure since a malicious script could get a response from the key without your permission.**
```
ykman otp chalresp --generate 2
ykpamcfg -2
```

### Require yubikey for system-wide authentication
This will require a yubikey present for graphical (lightdm) logins, non-graphical logins on TTYs, as well as `sudo`:

```
echo "auth    required   pam_yubico.so mode=challenge-response" | sudo tee -a /etc/pam.d/common-auth
```

### Full disk encryption setup

Install the `yubikey-luks` package:
```
sudo apt install yubikey-luks
```
If you are unsure of which device that contains your LUKS partition, run `lsblk`. In the below example, the device is `/dev/sda3`:
```
$ lsblk
NAME                                        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                                           8:0    1 119,3G  0 disk
├─sda1                                        8:1    1   512M  0 part  /boot/efi
├─sda2                                        8:2    1   732M  0 part  /boot
└─sda3                                        8:3    1   118G  0 part
  └─sdb3_crypt                              253:0    0   118G  0 crypt
    ├─elementary--vg-root                   253:1    0   117G  0 lvm   /
    └─elementary--vg-swap_1                 253:2    0   976M  0 lvm   [SWAP]
nvme0n1                                     259:0    0 953,9G  0 disk
├─nvme0n1p1                                 259:1    0   512M  0 part
├─nvme0n1p2                                 259:2    0 937,5G  0 part
└─nvme0n1p3                                 259:3    0  15,9G  0 part
```
List key slots for your LUKS device (only slot 0 should be used by default)
```
sudo cryptsetup luksDump /dev/sda3
```
Configure slot 1 for yubikey number 1:
```
sudo yubikey-luks-enroll -d /dev/sda3 -s 1
```
Configure slot 2 for yubikey number 2:
```
sudo yubikey-luks-enroll -d /dev/sda3 -s 2
```
Now restart your computer. Once verified that the yubikey can be used to unlock the disk, remove the default slot for the static password (slot 0):
```
sudo cryptsetup -q luksKillSlot /dev/sda3 0
```


### Lock the screen when yubikey is removed
Install the `finger` package:
```
sudo apt install finger
```
Create the `yubikey-removed-script`:
```
cat << 'EOF' | sudo tee /usr/local/bin/yubikey-removed-script
#!/usr/bin/env bash
# Locks the screen if a Yubikey is not plugged in

if ! env | grep DEVNAME; then
  exit 0;
fi

getXuser() {
  user=`finger | grep -m1 ":$displaynum " | awk '{print $1}'`
  if [ x"$user" = x"" ]; then
    user=`finger| grep -m1 ":$displaynum" | awk '{print $1}'`
  fi
  if [ x"$user" != x"" ]; then
    userhome=`getent passwd $user | cut -d: -f6`
    export XAUTHORITY=$userhome/.Xauthority
  else
    export XAUTHORITY=""
  fi
}

if [ -z "$(lsusb | grep Yubico)" ] ; then
  for x in /tmp/.X11-unix/*; do
      displaynum=`echo $x | sed s#/tmp/.X11-unix/X## | sed s#/##`
      getXuser
      if [ x"$XAUTHORITY" != x"" ]; then
          # extract current state
          export DISPLAY=":$displaynum"
      fi
  done

  enable_timer=true

  if [ "${enable_timer}" == "true" ]; then
    for i in {1..5}; do
      sleep 1;
      echo $(( $i*20 ));
    done | zenity --title "Yubikey removed" --text "Locking the screen\n(Press Escape to cancel)" --width 400 --progress --no-cancel --auto-close
    answer=$?
  fi

  if [[ "${enable_timer}" != "true" ]] || [[ ${answer} -eq 0 ]]; then
    export grep -z DBUS_SESSION_BUS_ADDRESS /proc/$(pidof light-locker)/environ
    logger "YubiKey Removed - Locking Workstation"
    su $user -c "/usr/bin/light-locker-command -l"
  fi
fi

EOF
```

Make it executable:
```
sudo chmod +x /usr/local/bin/yubikey-removed-script
```

Create a udev rule to trigger the `yubikey-removed-script`:
```
cat <<EOF | sudo tee /etc/udev/rules.d/yubikey-screenlock.rules
ACTION=="remove", ENV{ID_VENDOR}=="Yubico", RUN+="/usr/local/bin/yubikey-removed-script"
EOF
```

## U2F setup
Download https://raw.githubusercontent.com/Yubico/libu2f-host/master/70-u2f.rules and put it in `/etc/udev/rules.d/70-u2f.rules`. Reboot your computer and then head over to https://demo.yubico.com/u2f and follow the instructions. After the process, your browser will be capable of using U2F authentication with a number of services.

For Github, you can add your Yubikeys as U2F devices at https://github.com/settings/two_factor_authentication/configure after you first have enabled generic 2-factor authentication using a mobile app such as Google Authenticator, see https://help.github.com/articles/configuring-two-factor-authentication/#configuring-two-factor-authentication-using-fido-u2f

## GPG/SSH setup

Install the `opensc` package:
```
sudo apt install opensc -y
```
For each yubikey: change the default PIN+PUK:
```
yubico-piv-tool --action change-pin
yubico-piv-tool --action change-puk
```

Generate a public RSA certificate:
```
yubico-piv-tool --slot 9a --action generate -o public.pem
```

Generate a self-signed RSA certificate:
```
CERT_CANONICAL_NAME="${USER}'s SSH Key"
PUBLIC_KEY_FILE="public.pem"
OUTPUT_FILENAME="private.pem"
yubico-piv-tool \
  --action verify-pin \
  --action selfsign-certificate \
  --slot 9a \
  --subject "/CN=$CERT_CANONICAL_NAME/" \
  --input $PUBLIC_KEY_FILE \
  --output $OUTPUT_FILENAME
```

Import the newly created .pem file:
```
yubico-piv-tool \
  --action import-certificate \
  --slot 9a \
  --input private.pem \
  --touch-policy cached
```

Check the status:
```
yubico-piv-tool --action status
```

Add the PKCS11Provider to the SSH config on the client side:
```
echo "PKCS11Provider /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so" | sudo tee -a /etc/ssh/ssh_config
```

## On the server (Ubuntu)
Install required packages:
```
sudo apt install software-properties-common  # provides add-apt-repository
sudo add-apt-repository ppa:yubico/stable -y
sudo apt update
sudo apt install libpam-yubico -y
```
Enable `ChallengeResponseAuthentication` and PAM usage in `/etc/ssh/sshd_config`:
```
ChallengeResponseAuthentication yes
UsePAM yes
```

Create the SSH authfile:
```
key1=$(echo <touch first yubikey> | cut -c -12)
key2=$(echo <touch second yubikey> | cut -c -12)
echo "vagrant:${key1}:${key2}" | sudo tee /etc/ssh/authorized_yubikeys
```
### Configure the SSH daemon
Note: get API keys from https://upgrade.yubico.com/getapikey/ (Needed for configuring the server side)
```
Yubikey 1
Client ID:	<client id 1>
Secret key:	<secret key 1>

Yubikey 2
Client ID:	<client id 2>
Secret key:	<secret key 2>
```
Using the API keys above, add an entry to the `key_map` for each Yubikey. If you want
1-factor authentication with only the Yubikey present instead of 2-factor authentication (SSH key + YubiKey present), set REQUIRED_OR_SUFFICIENT
to "sufficient" instead of "required".
```
declare -A key_map
key_map["<client id 1>"]="secret key 1>"
key_map["<client id 2>"]="secret key 2>"
REQUIRED_OR_SUFFICIENT="required"
FILE="/etc/pam.d/sshd"
for key in ${!key_map[@]}; do
    secret=${key_map[${key}]}
    LINE="auth $REQUIRED_OR_SUFFICIENT pam_yubico.so id=$key key=$secret authfile=/etc/ssh/authorized_yubikeys"
    sudo sed -i "1s|^|$LINE \n|"  $FILE
done
```

### Links:
https://support.yubico.com/support/solutions/articles/15000011355-ubuntu-linux-login-guide-challenge-response

https://dgunter.com/2018/09/30/securing-linux-full-disk-encryption-with-a-multi-factor-hardware-token/

https://blog.programster.org/yubikey-cheatsheet

https://www.linode.com/docs/security/authentication/how-to-use-yubikey-for-two-factor-ssh-authentication/
