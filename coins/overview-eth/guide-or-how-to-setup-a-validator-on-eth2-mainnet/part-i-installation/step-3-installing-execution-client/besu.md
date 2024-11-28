# Besu

## Overview

{% hint style="info" %}
**Hyperledger Besu** es un cliente de Ethereum de código abierto diseñado para aplicaciones empresariales exigentes que requieren un procesamiento de transacciones seguro y de alto rendimiento en una red privada. Está desarrollado bajo la licencia Apache 2.0 y escrito en **Java**.
{% endhint %}

#### Enlaces Oficiales

| Tema          | Link                                                                                         |
| ------------- | -------------------------------------------------------------------------------------------- |
| Lanzamientos  | [https://github.com/hyperledger/besu/releases](https://github.com/hyperledger/besu/releases) |
| Documentación | [https://besu.hyperledger.org](https://besu.hyperledger.org/en/stable/)                      |
| Sitio Web     | [https://www.hyperledger.org/use/besu](https://www.hyperledger.org/use/besu)                 |

### 1. Configuración inicial

Cree un usuario de servicio para el servicio de ejecución, cree un directorio de datos y asigne la propiedad.

```bash
sudo adduser --system --no-create-home --group execution
sudo mkdir -p /var/lib/besu
sudo chown -R execution:execution /var/lib/besu
```

Instalar dependencias.

```bash
sudo apt install -y openjdk-21-jdk libjemalloc-dev jq
```

### 2. Instalar Binarios

* Descargar archivos binarios suele ser más rápido y cómodo.
* Construir a partir de código fuente puede ofrecer una mejor compatibilidad y está más alineado con el espíritu de FOSS (software gratuito de código abierto).

<details>

<summary>Option 1 - Download binaries</summary>

Ejecute lo siguiente para descargar automáticamente la última versión de Linux, descomprimir y limpiar.

```bash
RELEASE_URL="https://api.github.com/repos/hyperledger/besu/releases/latest"
TAG=$(curl -s $RELEASE_URL | jq -r .tag_name)
BINARIES_URL="https://github.com/hyperledger/besu/releases/download/$TAG/besu-$TAG.tar.gz"

echo Descargando URL: $BINARIES_URL

cd $HOME
wget -O besu.tar.gz $BINARIES_URL
tar -xzvf besu.tar.gz -C $HOME
rm besu.tar.gz
sudo mv $HOME/besu-* besu
```

Instalar los binarios.

<pre class="language-bash"><code class="lang-bash"><strong>sudo mv $HOME/besu /usr/local/bin/besu
</strong></code></pre>

</details>

<details>

<summary>Option 2 - Construir desde el código fuente</summary>

Construye los binarios.

```bash
mkdir -p ~/git
cd ~/git
# Clone the repo
git clone https://github.com/hyperledger/besu.git
cd besu
# Get new tags
git fetch --tags
# Get latest tag name
latestTag=$(git describe --tags `git rev-list --tags --max-count=1`)
# Checkout latest tag
git checkout $latestTag
# Build
./gradlew installDist
```

Verifique que Besu se haya creado correctamente comprobando la versión.

```shell
./build/install/besu/bin/besu --version
```

Salida de muestra de una versión compatible.

```
besu/v23.4.0/linux-x86_64/openjdk-java-17
```

Instale los binarios.

<pre class="language-shell"><code class="lang-shell"><strong>sudo cp -a $HOME/git/besu/build/install/besu /usr/local/bin/besu
</strong></code></pre>

</details>

### **3. Instalar y configurar systemd**

Crea un **systemd archivo unitario** para definir tu `execution.service` configuracion.

```bash
sudo nano /etc/systemd/system/execution.service
```

Pegue la siguiente configuración en el archivo.

```shell
[Unit]
Description=Besu Execution Layer Client service for Mainnet
Wants=network-online.target
After=network-online.target
Documentation=https://www.coincashew.com

[Service]
Type=simple
User=execution
Group=execution
Restart=on-failure
RestartSec=3
KillSignal=SIGINT
TimeoutStopSec=900
Environment="JAVA_OPTS=-Xmx5g"
ExecStart=/usr/local/bin/besu/bin/besu \
  --network=mainnet \
  --p2p-port=30303 \
  --rpc-http-port=8545 \
  --engine-rpc-port=8551 \
  --max-peers=25 \
  --metrics-enabled=true \
  --metrics-port=6060 \
  --rpc-http-enabled=true \
  --sync-mode=SNAP \
  --data-storage-format=BONSAI \
  --Xbonsai-limit-trie-logs-enabled=true \
  --data-path="/var/lib/besu" \
  --Xplugin-rocksdb-high-spec-enabled \
  --engine-jwt-secret=/secrets/jwtsecret
  
[Install]
WantedBy=multi-user.target
```

{% hint style="info" %}
**¿Tiene menos de 32 GB de RAM?** Eliminando la bandera, `--Xplugin-rocksdb-high-spec-enabled` es aconsejable.
{% endhint %}

Para salir y guardar, presiona `Ctrl` + `X`, luego `Y`,luego `Enter`.

Ejecute lo siguiente para habilitar el inicio automático en el momento del arranque.

```bash
sudo systemctl daemon-reload
sudo systemctl enable execution
```

Finalmente, inicie su cliente de capa de ejecución y verifique su estado.

```bash
sudo systemctl start execution
sudo systemctl status execution
```

Presione `Ctrl` + `C` para salir del estado.

### 4. Comandos útiles del cliente de ejecución

{% tabs %}
{% tab title="View Logs" %}
```bash
sudo journalctl -fu execution | ccze
```

Un cliente de ejecución **Besu** que funcione correctamente indicará "Fork-Choice-Updates". Por ejemplo,

```
2022-03-19 04:09:36.315+00:00 | vert.x-worker-thread-0 | INFO  | EngineForkchoiceUpdated | Consensus fork-choice-update: head: 0xcd2a_8b32..., finalized: 0xfa22_1142...
2022-03-19 04:09:48.328+00:00 | vert.x-worker-thread-0 | INFO  | EngineForkchoiceUpdated | Consensus fork-choice-update: head: 0xff1a_f12a..., finalized: 0xfa22_1142...
```
{% endtab %}

{% tab title="Stop" %}
```bash
sudo systemctl stop execution
```
{% endtab %}

{% tab title="Start" %}
```bash
sudo systemctl start execution
```
{% endtab %}

{% tab title="View Status" %}
```bash
sudo systemctl status execution
```
{% endtab %}

{% tab title="Reset Database" %}
Las razones comunes para restablecer la base de datos pueden incluir:

*  Recuperación de una base de datos dañada debido a un corte de energía o una falla de hardware
* Resincronización para reducir el uso de espacio en disco
* Actualización a un nuevo formato de almacenamiento.

```bash
sudo systemctl stop execution
sudo rm -rf /var/lib/besu/*
sudo systemctl restart execution
```

Time to re-sync the execution client can take a few hours up to a day.
{% endtab %}
{% endtabs %}

Ahora que su cliente de ejecución está configurado e iniciado, continúe con el siguiente paso para configurar su cliente de consenso.

{% hint style="warning" %}
Si está revisando los registros y ve alguna advertencia o error, tenga paciencia, ya que normalmente se resolverán una vez que tanto su cliente de ejecución como el de consenso estén completamente sincronizados con la red Ethereum.
{% endhint %}
