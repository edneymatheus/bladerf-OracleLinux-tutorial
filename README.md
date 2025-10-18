# Tutorial: Instalando e Configurando o **bladeRF** no **Oracle Linux 8.10** (RHEL-like)

Guia passo-a-passo para compilar, instalar e testar o **bladeRF** em Oracle Linux 8.10, incluindo configuração de repositórios, dependências, udev, CLI, firmware/FPGA e bindings Python. No fim há uma seção opcional para acesso remoto via USB-over-IP.

---

## Índice
1. [Pré-requisitos](#pré-requisitos)  
2. [Preparando o sistema (repositórios + update)](#preparando-o-sistema)  
3. [Instalando dependências de build](#dependencias)  
4. [Compilando e instalando o `libbladeRF` (+ CLI)](#build-host)  
5. [Udev e permissões USB](#udev)  
6. [Firmware e FPGA](#firmware-fpga)  
7. [Testes no CLI](#testes-cli)  
8. [Usando o bladeRF com Python](#python)  
9. [Problemas comuns (Oracle Linux)](#troubleshooting)  
10. [Acesso remoto (USB-over-IP) — opcional](#usbip)  
11. [Conclusão](#conclusao)  
12. [Referências](#referencias)

---

## 1) Pré-requisitos <a name="pré-requisitos"></a>
- **Oracle Linux 8.10** (ou compatível RHEL8) com acesso `sudo`.
- Porta **USB 3.0** e **cabo USB 3.0**.
- Placa **bladeRF** (1.x ou 2.x).
- Internet para baixar dependências e imagens de firmware/FPGA.

---

## 2) Preparando o sistema (repositórios + update) <a name="preparando-o-sistema"></a>

```bash
# atualizar metadados e pacotes
sudo dnf makecache --refresh
sudo dnf upgrade -y

# plugins e repositórios úteis
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --set-enabled ol8_codeready_builder || true
sudo dnf install -y oracle-epel-release-el8
sudo dnf config-manager --set-enabled ol8_developer_EPEL || true
sudo dnf makecache --refresh
```

> Muitos pacotes *-devel* e extras vêm do CodeReady/Developer EPEL.

---

## 3) Instalando dependências de build <a name="dependencias"></a>

```bash
# toolchain equivalente ao "build-essential"
sudo dnf groupinstall -y "Development Tools"

# bibliotecas necessárias
sudo dnf install -y \
  cmake git pkgconf-pkg-config \
  libusbx-devel libcurl-devel glib2-devel ncurses-devel readline-devel
```

> **Nota sobre `libtecla`:** não existe pacote `libtecla[-devel]` nos repositórios oficiais do OL8.  
> O `bladeRF-cli` funciona **sem** libtecla (perde apenas autocompletar avançado). Se você **precisar**, veja como compilar do fonte em [Problemas comuns](#troubleshooting).

---

## 4) Compilando e instalando o `libbladeRF` (+ CLI) <a name="build-host"></a>

```bash
cd ~
git clone https://github.com/Nuand/bladeRF.git
cd bladeRF
# opcional, mas seguro se também for usar firmware/FPGA do mesmo repo
git submodule update --init --recursive

cd host
mkdir build && cd build

# Opção A (recomendada): instalar em /usr (evita mexer no ldconfig)
cmake .. -DCMAKE_BUILD_TYPE=Release -DENABLE_LIBTECLA=OFF -DCMAKE_INSTALL_PREFIX=/usr

# Opção B (padrão /usr/local): se usar, configure o ldconfig depois (abaixo)
# cmake .. -DCMAKE_BUILD_TYPE=Release -DENABLE_LIBTECLA=OFF

make -j"$(nproc)"
sudo make install
sudo ldconfig
```

Se você instalou em **/usr/local**, torne a lib visível permanentemente:

```bash
echo -e "/usr/local/lib64\n/usr/local/lib" | sudo tee /etc/ld.so.conf.d/bladerf-usrlocal.conf
sudo ldconfig
```

---

## 5) Udev e permissões USB <a name="udev"></a>

Regra simples para usar o device sem `sudo`:

```bash
sudo tee /etc/udev/rules.d/88-nuand.rules >/dev/null <<'EOF'
SUBSYSTEM=="usb", ATTR{idVendor}=="2cf0", MODE="0666", TAG+="uaccess"
EOF

sudo udevadm control --reload
sudo udevadm trigger
# reconecte o cabo USB da bladeRF após aplicar a regra
```

> Alternativa baseada em grupo:
> ```bash
> sudo groupadd -f plugdev
> sudo usermod -aG plugdev $USER
> # e na regra use:  GROUP="plugdev", MODE="0664"
> # faça logout/login (ou: newgrp plugdev)
> ```

---

## 6) Firmware e FPGA <a name="firmware-fpga"></a>

Baixe as imagens atuais no site da Nuand:
- **Firmware (FX3)**: `bladeRF_fw_vX.Y.Z.img`  
- **FPGA** (ex.: `hostedxA4_v0.16.0.rbf` para micro xA4)

Atualizar / carregar:

```bash
# carregar FPGA em RAM (para a sessão atual)
bladeRF-cli -l ./hostedxA4_v0.16.0.rbf

# gravar FPGA na flash para autoload
bladeRF-cli -L ./hostedxA4_v0.16.0.rbf

# atualizar firmware (se necessário)
bladeRF-cli -f ./bladeRF_fw_v2.5.0.img
```

---

## 7) Testes no CLI <a name="testes-cli"></a>

```bash
# o kernel viu o device?
sudo dnf install -y usbutils
lsusb | egrep -i '2cf0|nuand'    # deve listar 2cf0:5250/5251

# listar devices via CLI
bladeRF-cli -p

# modo interativo
bladeRF-cli -i
# dentro do CLI:
#   version
#   info
#   help
#   quit
```

Exemplo de captura rápida:

```bash
bladeRF-cli -i
# set frequency rx 100M
# set samplerate rx 10M
# rx config file=/tmp/samples.sc16 format=bin n=1000000
# rx start
# rx wait
# quit
ls -lh /tmp/samples.sc16
```

---

## 8) Usando o bladeRF com Python <a name="python"></a>

### 8.1 Escolha do Python
OL8 vem com **Python 3.6**; funciona, mas é recomendável **Python 3.9** via AppStream:

```bash
sudo dnf module enable -y python39
sudo dnf install -y python39 python39-devel python39-pip
python3.9 -V
```

### 8.2 Criar e ativar o *venv*
```bash
python3.9 -m venv ~/bladerf_env   # use python3 se quiser ficar no 3.6
source ~/bladerf_env/bin/activate
python -m pip install --upgrade pip setuptools wheel
```

### 8.3 Instalar bindings Python
> Requer o `libbladeRF` já instalado e visível via `ldconfig`.

```bash
# garantir que o pkg-config enxerga o .pc (se instalou em /usr/local)
export PKG_CONFIG_PATH=/usr/local/lib64/pkgconfig:/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH

# dependências Python
python -m pip install cython numpy matplotlib

# instalar os bindings
cd ~/bladeRF/host/libraries/libbladeRF_bindings/python
python -m pip install .
```

### 8.4 Teste de import (sem device)
```python
# salve como test_bladerf.py e execute com o venv ativo
try:
    import bladerf
    print("Bindings OK.")
    from bladerf import BladeRF
    try:
        dev = BladeRF()       # falhará se não houver placa conectada
        print("Device aberto!")
        dev.close()
    except Exception as e:
        print("Sem device (esperado se não há placa):", e)
except Exception as e:
    print("Falha ao importar 'bladerf':", e)
```

---

## 9) Problemas comuns (Oracle Linux) <a name="troubleshooting"></a>

- **`error while loading shared libraries: libbladeRF.so.2`**  
  A instalação foi para `/usr/local/lib(64)` e o linker não sabe disso.  
  Soluções:
  ```bash
  echo -e "/usr/local/lib64\n/usr/local/lib" | sudo tee /etc/ld.so.conf.d/bladerf-usrlocal.conf
  sudo ldconfig
  # ou reinstale com: -DCMAKE_INSTALL_PREFIX=/usr
  ```

- **`bladeRF-cli -p` → No devices are available**  
  Verifique: `lsusb` → IDs `2cf0:5250/5251`.  
  Se não aparecer, confira cabo/porta/energia. Se aparecer, é permissão: aplique a regra **udev** e reconecte.

- **`Could NOT find ...` (CURL/USB/GLib) no CMake**  
  Garanta os *-devel*: `libcurl-devel`, `libusbx-devel`, `glib2-devel`, `ncurses-devel`.

- **`pkg-config` não encontra o libbladeRF**  
  Exporte `PKG_CONFIG_PATH` com `/usr/local/lib64/pkgconfig` (ou instale com `-DCMAKE_INSTALL_PREFIX=/usr`).

- **Autocompletar do CLI (libtecla) ausente**  
  O OL8 não fornece `libtecla`. Opcional: compilar do fonte:
  ```bash
  sudo dnf install -y ncurses-devel
  cd /tmp
  curl -LO https://www.astro.caltech.edu/~mcs/tecla/libtecla-1.6.3.tar.gz
  tar xzf libtecla-1.6.3.tar.gz && cd libtecla-1.6.3
  ./configure --prefix=/usr
  make -j"$(nproc)"
  sudo make install
  sudo ldconfig
  # reconfigure/rebuild o bladeRF SEM -DENABLE_LIBTECLA=OFF
  ```

---

## 10) Acesso remoto (USB-over-IP) — opcional <a name="usbip"></a>

### Host **Windows** (placa conectada)
No Windows (PowerShell Admin):
```powershell
winget install usbipd
usbipd list
usbipd bind --busid <BUSID>
usbipd share --busid <BUSID> --tcp-port 3240
# abra a porta no firewall se necessário
```

No Oracle Linux (cliente):
```bash
sudo dnf install -y usbip
sudo modprobe vhci-hcd
usbip list -r <IP_WINDOWS>
sudo usbip attach -r <IP_WINDOWS> -b <BUSID>
lsusb
bladeRF-cli -p
```

### Host **Linux** (placa conectada)
No Linux servidor:
```bash
sudo dnf install -y usbip
sudo modprobe usbip-host
sudo usbipd -D
usbip list -l     # anote o busid (ex.: 1-2)
```
No Oracle Linux (cliente):
```bash
sudo modprobe vhci-hcd
usbip list -r <IP_SERVIDOR>
sudo usbip attach -r <IP_SERVIDOR> -b <BUSID>
bladeRF-cli -p
```

> Para desfazer: `sudo usbip detach -p <porta>` no cliente.

---

## 11) Conclusão <a name="conclusao"></a>
Com este guia você:
- Preparou repositórios e dependências no **Oracle Linux 8.10**.  
- Compilou e instalou **libbladeRF** e **bladeRF-cli**.  
- Configurou **udev** e (opcionalmente) **firmware/FPGA**.  
- Testou no **CLI** e habilitou **bindings Python** com *venv*.  
- (Opcional) Habilitou acesso remoto via **USB-over-IP**.

---

## 12) Referências <a name="referencias"></a>
- Nuand — bladeRF (repositório): https://github.com/Nuand/bladeRF  
- Imagens de firmware/FPGA: https://www.nuand.com/  
- PySDR — Using the bladeRF: https://pysdr.org/content/bladerf.html  
- usbipd-win (Windows USB/IP): https://github.com/dorssel/usbipd-win

> **Boas práticas:** não use `sudo` dentro do diretório do código (exceto no `make install`). Se precisar remover o repo e surgir “Permission denied”, acerte a posse: `sudo chown -R $USER:$USER ~/bladeRF` e só então `rm -rf`.
