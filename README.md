# gentoo_secureboot_dellt5820
A step by step process for getting secure boot working in Gentoo amd64 on a Dell Precision T5820.

## Dependencies
'''
emerge -v app-crypt/efitools app-crypt/sbsigntools sys-boot/mokutil dev-libs/openssl
'''

## Disable Secure Boot on BIOS
First make sure SecureBoot is disabled in BIOS

## Setup /etc/efikeys
'''
mkdir -p /etc/efikeys
chmod -v 700 /etc/efikeys
cd /etc/efikeys

## Backup Current Keys
Make sure you save the keys already installed in the system. You can do this via the BIOS or with 'efi-readvar'.

'''
efi-readvar -v PK -o old_PK.esl
efi-readvar -v KEK -o old_KEK.esl
efi-readvar -v db -o old_db.esl
efi-readvar -v dbx -o old_dbx.esl
'''

## Generate New Keys
'''
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=your platform key/" -keyout PK.key -out PK.crt -days 3650 -nodes -sha256 
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=your key-exchange-key/" -keyout KEK.key -out KEK.crt -days 3650 -nodes -sha256 
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=your kernel-signing key/" -keyout db.key -out db.crt -days 3650 -nodes -sha256 
chmod -v 400 *.key
'''

## Preparing Keystore - Update Files Using Keys
'''
cert-to-efi-sig-list -g "$(uuidgen)" PK.crt PK.esl 
sign-efi-sig-list -k PK.key -c PK.crt PK PK.esl PK.auth 

cert-to-efi-sig-list -g "$(uuidgen)" KEK.crt KEK.esl 
sign-efi-sig-list -a -k PK.key -c PK.crt KEK KEK.esl KEK.auth 

cert-to-efi-sig-list -g "$(uuidgen)" db.crt db.esl 
sign-efi-sig-list -a -k KEK.key -c KEK.crt db db.esl db.auth 

sign-efi-sig-list -k KEK.key -c KEK.crt dbx old_dbx.esl old_dbx.auth 
'''

### Create DER versions
'''
openssl x509 -outform DER -in PK.crt -out PK.cer 
openssl x509 -outform DER -in KEK.crt -out KEK.cer 
openssl x509 -outform DER -in db.crt -out db.cer 
'''

## Create Compound Keys - Combining with current keys
'''
cat old_KEK.esl KEK.esl > compound_KEK.esl 
cat old_db.esl db.esl > compound_db.esl 
sign-efi-sig-list -k PK.key -c PK.crt KEK compound_KEK.esl compound_KEK.auth 
sign-efi-sig-list -k KEK.key -c KEK.crt db compound_db.esl compound_db.auth 
'''

Back these keys up to a USB drive for loading into the BIOS via BIOS Setup.

### Using Your Own Keys - Removing current keys
Only do this if you're okay replacing all the current keys on the BIOS
'''
cp -v KEK.esl compound_KEK.esl 
cp -v db.esl compound_db.esl 
sign-efi-sig-list -k PK.key -c PK.crt KEK compound_KEK.esl compound_KEK.auth 
sign-efi-sig-list -k KEK.key -c KEK.crt db compound_db.esl compound_db.auth 
'''

## Enter BIOS Setup and Clear Current Keys
Reboot the machine and enter the BIOS Setup Utility.
On the Dell T5820 I recommend loading the keys directly via the BIOS Setup Utility. 
The BIOS Setup Utility will allow you to overwrite the current keys while checking for the proper format, preventing you from using a key that could break your BIOS.
Use the keys saved to a USB drive to overwrite the current keys in the proper order
1. dbx
2. db
3. KEK
4. PK

### Setting Keys with efi-updatevar
I recommend against this unless you know exactly what key format your BIOS accepts. At all possible, you should try to load your keys via the BIOS>
'''
efi-updatevar -e -f old_dbx.esl dbx 
efi-updatevar -e -f compound_db.esl db 
efi-updatevar -e -f compound_KEK.esl KEK 
efi-updatevar -f PK.auth PK 
'''

## Verify Keys with efi-readvar
Boot back into Gentoo and verify the right keys were loaded using 'efi-readvar'.
'''
efi-readvar
'''

## Make sure Grub is properly installed
Grub needs to be installed properly for SecureBoot to work.

'''
mount /boot
grub-install --target=x86_64-efi --efi-directory=/boot --modules="normal tpm" --disable-shim-lock
'''

## Sign Grub efi and vmlinuz
'''
cd /boot/EFI/gentoo
mv grubx64.efi grubx64-unsigned.efi
sbsign --key /etc/efikeys/db.key --cert /etc/efikeys/db.crt grubx64-unsigned.efi --output grubx64.efi

cd /boot
mv vmlinuz-[KERNEL-VERSION] vmlinuz-[KERNELVERSION].unsigned
sbsign --key /etc/efikeys/db.key --cert /etc/efikeys/db.crt vmlinuz-[KERNEL-VERSION].unsigned --output vmlinuz-[KERNEL-VERSION]
'''

## Reboot with SecureBoot Disabled
Reboot the machine. Make sure you boot the signed 'grubx64.efi' to test.
Once the machine boots just fine using the signed 'grubx64.efi' and 'vmlinuz' then you're good to move onto the next step.

## Reboot with SecureBoot Enabled
Reboot the machine, enabling secure boot.


