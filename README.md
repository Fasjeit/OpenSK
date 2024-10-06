# <img alt="OpenSK logo" src="docs/img/OpenSK.svg" width="200px">

update - не смог сейчас даже u2f bootloader засунуть на mdk. перешёл на nordic

update2 - починил. корректная команда обновленя на u2f - 

```
nrfutil dfu serial -pkg uf2_bootloader-0.2.13-44-gb2b4284-nosd_signed.zip -p COM7
```

полностью снёс nrfutil и поставил последний. Заработало!


*************

mdk

toolchaing setup
https://wiki.makerdiary.com/nrf52840-mdk-usb-dongle/guides/opensk/building/#building-opensk

reset storage mdk

```
./deploy.py --board=nrf52840_mdk_dfu --programmer=none --erase_storage
uf2conv -c -f 0xada52840 -o target/opensk.uf2 target/nrf52840_mdk_dfu_merged.hex
```

build mdk 

```
./deploy.py --board=nrf52840_mdk_dfu --programmer=none --opensk
uf2conv -c -f 0xada52840 -o target/opensk.uf2 target/nrf52840_mdk_dfu_merged.hex
```

nordic

reset storage nordic

```
./deploy.py --board=nrf52840_dongle_dfu --erase_storage --programmer=none

```
build

```
./deploy.py --board=nrf52840_dongle_dfu --opensk --programmer=none
```

just write hex via app

*************

инстуркции 
https://github.com/google/OpenSK/blob/stable/docs/boards/nrf52840_dongle.md

https://github.com/google/OpenSK/blob/stable/docs/install.md

https://github.com/google/OpenSK/blob/develop/docs/install.md

сборка

```
./reset.sh
./setup.sh
```

Сорка очищалки памяти
```
./deploy.py --board=nrf52840_dongle_dfu --programmer=none --erase_storage
```

Сборка самой штуки
```
./deploy.py --board=nrf52840_dongle_dfu --opensk --programmer=none
```

После сборки очищалки\штуки просто залить hex через nrf connect programmer. Не забыть нажимать clear file чтоб не было наслоение файлов при прошивке.

Посмотреть добавленные ключи на токене можно через chrome на linux, или через скрипт

<details>
    <summary>fido_creds.py</summary>
    
    ```python
    from fido2.hid import CtapHidDevice
    from fido2.ctap2 import CredentialManagement, Ctap2, ClientPin
    from fido2.ctap2.bio import BioEnrollment

    
    class Ctap2Node:
        def __init__(self, connection):
            super().__init__()
            self.ctap = Ctap2(connection)
            self._info = self.ctap.info
            self.client_pin = ClientPin(self.ctap)
            self._auth_blocked = False
            self._token = None
    
        def unlock(self, pin):
            permissions = ClientPin.PERMISSION(0)
            if CredentialManagement.is_supported(self._info):
                permissions |= ClientPin.PERMISSION.CREDENTIAL_MGMT
            if BioEnrollment.is_supported(self._info):
                permissions |= ClientPin.PERMISSION.BIO_ENROLL
            self._token = self.client_pin.get_pin_token(pin, permissions)
    
        def cm(self):
            return CredentialManagement(
                self.ctap,
                pin_uv_protocol=self.client_pin.protocol,
                pin_uv_token=self._token,
            )
    
        def list_rp(self, cm: CredentialManagement):
            rps = cm.enumerate_rps()
            print(f'rps: {rps}')
    
    
    def main(pin):
        devices = list(CtapHidDevice.list_devices())
        device = Ctap2Node(devices[0])
        device.unlock(pin)
        cm = device.cm()
    
        # list rp
        enumerate_rps = cm.enumerate_rps()
        print(f'\n\nenumerate_rps: {enumerate_rps}')
        for index, rp in enumerate(enumerate_rps):
            print(f'  {index}:{rp[CredentialManagement.RESULT.RP]["id"]}')
        index = input('select rp:').strip()
        select_rp = enumerate_rps[int(index)]
    
        # list cred
        enumerate_creds = cm.enumerate_creds(rp_id_hash=select_rp[CredentialManagement.RESULT.RP.RP_ID_HASH])
        print(f"\n\nenumerate_creds: {enumerate_creds}")
        for index, cred in enumerate(enumerate_creds):
            print(f'  {index}:{cred[CredentialManagement.RESULT.USER]["name"]}')
        index = input('select cred:').strip()
        select_cred = enumerate_creds[int(index)]
    
        # delete cred
        cm.delete_cred(select_cred[CredentialManagement.RESULT.CREDENTIAL_ID])
    
    
    if __name__ == '__main__':
        main('12345678') # set pin
    ```
</details>

Всё что ниже - старое.


ПОРТ МОЖЕТ МЕНЯТЬСЯ ПОСЛЕ ПРОШИВОК! ПРОВЕРЯТЬ ПОРТ ПЕРЕД КАЖДОЙ ПРОШИВКОЙ!

Для работы необходимо заменить загрузчик на u2f.

Для этого нужно установить утилитку nrfutil

```
pip install nrfutil
nrfutil dfu usb-serial -pkg uf2_bootloader-0.2.13-44-gb2b4284-nosd_signed.zip -p <your-serial-port-name>
```

Имя порта в формате `COM<N>` для винды.
Ссылка на инсрукцию и бутлоадеры - https://github.com/makerdiary/nrf52840-mdk-usb-dongle/tree/master/firmware/open_bootloader

После обновления bootloader'а необходимо зайти в режим прошивки (зажать кнопку при коннекте или два ража нажать на кнопку после подключения (см проблемы).
Загрузить в корень открывшегося устройства opensk_nrf52840_mdk_usb_dongle_gece14d7.uf2 (ссылка - https://github.com/makerdiary/nrf52840-mdk-usb-dongle/tree/master/firmware/OpenSK)

После чего всё должно работать.

Ссылка на wiki - https://wiki.makerdiary.com/nrf52840-mdk-usb-dongle/opensk/getting-started/

Для того чтобы вернуть старый загрузчик нужна утилитка adafruit-nrfutil

```
pip3 install --user adafruit-nrfutil
adafruit-nrfutil --verbose dfu serial --package nrf52840_mdk_usb_dongle_open_bootloader_v1.2.0.uf2.zip -p <serial-port-name> -b 115200 --singlebank
```

Почему то путь не подцепился до adafruit-nrfutil. Указывал путь руками 
```
C:\Users\Fasjeit\AppData\Roaming\Python\Python36\Scripts\adafruit-nrfutil.exe --verbose dfu serial --package nrf52840_mdk_usb_dongle_open_bootloader_v1.2.0.uf2.zip -p COM4 -b 115200 --singlebank
```

## Проблемы
### После прошивки на u2f bootloader не входит в режим bootloader'а при нажатии на кнопку во время подключения
Решение - два раза нажать на кнопку. Ссылка - https://learn.adafruit.com/adafruit-feather-m0-express-designed-for-circuit-python-circuitpython/uf2-bootloader-details

### При нажатии на кнопку плата перезапускается
Проблема в том, что кнопку можно настроить на ребут, что и происходит при использовании примеров из Nordic SDK. Выставляется переменная, которая влияет на функционирование кнопки. В результате данных режим сохраняется даже при перепрошвке. Решение - прошить прошивкой, которая сбрасывает данный флаг.
ссылка - https://github.com/makerdiary/nrf52840-mdk-usb-dongle/issues/14
ещё ссылка - https://github.com/makerdiary/nrf52840-mdk-usb-dongle/issues/9
прошивка - https://github.com/makerdiary/nrf52840-mdk-usb-dongle/tree/master/examples/nrf5-sdk/pselreset_erase/hex

### Окирпичивание
Если удачно прошиться версией для nordiс токена, то кнопка будет ребутать устройство (из за различной распиновки) при нажатии. При это зайти в u2f загрузчик не будет возможно. Решение прошивка через вшешний программатор. Через rPi не получилось. Купил j-link, через него заработало.

Распиновка j-link 

![img](https://www.amebaiot.com.cn/wp-content/uploads/2019/07/start-2.png)

Распиновка makerDiary Токена
![img](https://wiki.makerdiary.com/nrf52840-mdk-usb-dongle/assets/images/nrf52840-mdk-usb-dongle-pinout.png)

Закинул u2f bootloader [отсюда](https://github.com/makerdiary/nrf52840-mdk-usb-dongle/tree/master/firmware/uf2_bootloader) через[j-flash lite](https://www.programmersought.com/article/43572405542/). Выбирал просто nrf52840 со стандартными параметрами. Hex файл залился, загрузчик работает.

[![Build Status](https://travis-ci.org/google/OpenSK.svg?branch=master)](https://travis-ci.org/google/OpenSK)

## OpenSK

This repository contains a Rust implementation of a
[FIDO2](https://fidoalliance.org/fido2/) authenticator.

We developed this as a [Tock OS](https://tockos.org) application and it has been
successfully tested on the following boards:

*   [Nordic nRF52840-DK](https://www.nordicsemi.com/Software-and-Tools/Development-Kits/nRF52840-DK)
*   [Nordic nRF52840-dongle](https://www.nordicsemi.com/Software-and-Tools/Development-Kits/nRF52840-Dongle)

## Disclaimer

This project is proof-of-concept and a research platform. It's still under
development and as such comes with a few limitations:

### FIDO2

Although we tested and implemented our firmware based on the published
[CTAP2.0 specifications](https://fidoalliance.org/specs/fido-v2.0-ps-20190130/fido-client-to-authenticator-protocol-v2.0-ps-20190130.html),
our implementation was not reviewed nor officially tested and doesn't claim to
be FIDO Certified.

### Cryptography

We're currently still in the process on making the
[ARM&reg; CryptoCell-310](https://developer.arm.com/ip-products/security-ip/cryptocell-300-family)
embedded in the
[Nordic nRF52840 chip](https://infocenter.nordicsemi.com/index.jsp?topic=%2Fps_nrf52840%2Fcryptocell.html)
work to get hardware-accelerated cryptography. In the meantime we implemented
the required cryptography algorithms (ECDSA, ECC secp256r1, HMAC-SHA256 and
AES256) in Rust as a placeholder. Those implementations are research-quality
code and haven't been reviewed. They don't provide constant-time guarantees and
are not designed to be resistant against side-channel attacks.

## Installation

For a more detailed guide, please refer to our
[installation guide](docs/install.md).

1.  If you just cloned this repository, run the following script (**Note**: you
    only need to do this once):

    ```shell
    ./setup.sh
    ```

2.  If you have a Nordic development board that doesn't already have Tock OS
    installed, you can install both TockOS and the OpenSK application by running
    the following command, depending on the board you have:

    ```shell
    # Nordic nRF52840-DK board
    board=nrf52840dk ./deploy.sh os app
    # Nordic nRF52840-Dongle
    board=nrf52840_dongle ./deploy.sh os app
    ```

3.  If Tock OS is already installed and you want to install/update the OpenSK
    application on your board (**Warning**: it will erase the locally stored
    credentials), run:

    ```shell
    ./deploy.sh app
    ```

4.  On Linux, you may want to avoid the need for `root` privileges to interact
    with the key. For that purpose we provide a udev rule file that can be
    installed with the following command:

    ```shell
    sudo cp rules.d/55-opensk.rules /etc/udev/rules.d/ &&
    sudo udevadm control --reload
    ```

### Customization

If you build your own security key, depending on the hardware you use, there are
a few things you can personalize:

1.  If you have multiple buttons, choose the buttons responsible for user
    presence in main.rs.
2.  Decide whether you want to use batch attestation. There is a boolean flag in
    `ctap/mod.rs`. It is mandatory for U2F, and you can create your own
    self-signed certificate. The flag is used for FIDO2 and has some privacy
    implications. Please check
    [WebAuthn](https://www.w3.org/TR/webauthn/#attestation) for more
    information.
3.  Decide whether you want to use signature counters. Currently, only global
    signature counters are implemented, as they are the default option for U2F.
    The flag in `ctap/mod.rs` only turns them off for FIDO2. The most privacy
    preserving solution is individual or no signature counters. Again, please
    check [WebAuthn](https://www.w3.org/TR/webauthn/#signature-counter) for
    documentation.
4.  Depending on your available flash storage, choose an appropriate maximum
    number of supported residential keys and number of pages in
    `ctap/storage.rs`.

### 3D printed enclosure

To protect and carry your key, we partnered with a professional designer and we
are providing a custom enclosure that can be printed on both professional 3D
printers and hobbyist models.

All the required files can be downloaded from
[Thingiverse](https://www.thingiverse.com/thing:4132768) including the STEP
file, allowing you to easily make the modifications you need to further
customize it.

## Contributing

See [Contributing.md](docs/contributing.md).
