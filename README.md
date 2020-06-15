# Tutorial Quarkus.io : Consume Kafka Topic and Stream as API using Server Sent Event (SSE) with Knative - Bahasa Indonesia

Tutorial ini akan melakukan hands on mengenai cara stream Kafka ke HTML dengan menggunakan metode Server Send Event (SSE) dengan opsi deploy dengan menggunakan serverless knative.
Kelebihan SSE dibandingkan dengan menggunakan Web Socket (ws://) adalah SSE menggunakan protokol http(s) dan satu arah, hanya dari server ke client saja.


Prerequsite tutorial ini adalah:
- Java 8 
- Maven **3.6.2 keatas** -> versi dibawah ini tidak didukung
- IDE kesayangan anda (RedHat CodeReady, Eclipse, VSCode, Netbeans, InteliJ, dll)
- Git client
- Dasar pemrograman Java
- Untuk build native image diperlukan Docker runtime
- Pre-installed Kafka cluster on Openshift 
- Optional: knative cli, RedHat Openshift Serverless Operator


## Clone code dari GitHub saya
Download source code dari repository [GitHub](https://github.com/erfinfeluzy/quarkus-kafka-consumer.git) saya:
```bash
> git clone https://github.com/erfinfeluzy/quarkus-kafka-consumer
> cd quarkus-kafka-consumer
> mvn quarkus:dev
```
> Note : Mode ini digunakan menggunakan mode *Development*, untuk menjalankan secara native dapat menggunakan


## Pembahasan Kode

### Library yang dibutuhkan
Tambahkan library kafka client pada file pom.xml
```xml
<dependency>
	<groupId>io.quarkus</groupId>
	<artifactId>quarkus-smallrye-reactive-messaging-kafka</artifactId>
</dependency>

...


```
> Note:
> 1. Quarkus menggunakan library dari [Smallrye.io](https://smallrye.io/) sebagai kafka client consumer.

### Konfigurasi Kafka Consumer
```properties
# Configure the Kafka source (we read from it)
kafka.bootstrap.servers=my-cluster-kafka-brokers:9092
mp.messaging.incoming.mytopic-subscriber.connector=smallrye-kafka
mp.messaging.incoming.mytopic-subscriber.topic=mytopic
mp.messaging.incoming.mytopic-subscriber.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```
> Note: **my-cluster-kafka-brokers:9092** adalah alamat kafka broker dalam Openshift saya

### Consume Topic Kafka dan teruskan ke Server Send Event emitter.
Snippet dibawah untuk subscribe ke topic **mytopic**, kemudian data dari topic diteruskan channel stream internal **my-internal-data-stream**.
```java
    @Incoming("mytopic-subscriber")
    @Outgoing("my-internal-data-stream")
    @Broadcast
    public String process(Message<String> incoming) {
    	
    	Long offset = getOffset(incoming);
    	
    	return "Kafka Offset=" + offset + "; message=" + incoming.getPayload();
    
    }
```
> Note:
> 1. **@Incoming("mytopic-subscriber")** : listen ke Kafka topic **mytopic** yang sudah dikonfigurasikan di application.properties
> 2. **@Outgoing("my-internal-data-stream") dan **@Broadcast** : message kafka yang diterima, diproses dengan menambahkan atribut offset number pada Kafka message, lalu diteruskan ke streaming endpoint via anotasi MicroProfile @Outgoing dan @Broadcast.


### Create Controller untuk melakukan stream SSE
Snippet berikut untuk melakukan stream data via http dengan menggunakan SSE pada endpoint **/stream**.
```java
	@Inject
	@Channel("my-internal-data-stream")
	Publisher<String> myDataStream;

	@GET
	@Path("/stream")
	@Produces(MediaType.SERVER_SENT_EVENTS)
	@SseElementType("text/plain")
	public Publisher<String> stream() {

		return myDataStream;
	}

```
Hasil dapat dilihat dengan menggunakan perintah curl:
```bash
> curl http://localhost:8080/stream
```
### Stream data ke HTML
Snippet javascript dibawah digunakan untuk menampilkan SSE event pada halaman html
```java
function initialize() {
	const eventSource = new EventSource('/stream');
	eventSource.onmessage = e => {
		const msg = e.data;
		document.getElementById("mycontent").innerHTML += "<br/>" + msg;
	};
	eventSource.onopen = e => console.log('open');
	eventSource.onerror = e => {
		if (e.readyState == EventSource.CLOSED) {
			console.log('close');
		}
		else {
			console.log(e);
		}
	};
	eventSource.addEventListener('second', function(e) {
		console.log('second', e.data);
	}, false);
}
window.onload = initialize;
```

## Voila! Kamu sudah berhasil
Coba di browser pada url berikut [http://$OCP_ROUTE:8080](http://$OCP_ROUTE:8080).

## Bonus! Deploy aplikasi kamu ke Red Hat Openshift

### Step 1: Build aplikasi sebagai native container
```bash
$ mvn clean package -Dnative -Dquarkus.native.container-build=true
```
Perintah ini akan generate aplikasi yang secara native run as container. setelah langkah ini cek file ** \*runner\* ** di foler target
```bash
$ ls -altr target/
...
-rw-r--r--  1 erfinfeluzy  staff      4509 Jun 15 08:15 quarkus-kafka-consumer-1.0-SNAPSHOT.jar
drwxr-xr-x  3 erfinfeluzy  staff        96 Jun 15 08:15 generated-sources
drwxr-xr-x  3 erfinfeluzy  staff        96 Jun 15 08:15 maven-archiver
drwxr-xr-x  3 erfinfeluzy  staff        96 Jun 15 08:15 maven-status
drwxr-xr-x  5 erfinfeluzy  staff       160 Jun 15 08:15 classes
-rwxr-xr-x  1 erfinfeluzy  staff  41647448 Jun 15 08:18 quarkus-kafka-consumer-1.0-SNAPSHOT-runner
drwxr-xr-x  4 erfinfeluzy  staff       128 Jun 15 08:18 quarkus-kafka-consumer-1.0-SNAPSHOT-native-image-source-jar
drwxr-xr-x  9 erfinfeluzy  staff       288 Jun 15 08:27 .
drwxr-xr-x  9 erfinfeluzy  staff       288 Jun 15 10:46 ..
```

### Step 2: Build aplikasi menjadi container
```bash
$ docker build -f src/main/docker/Dockerfile.native -t quarkus/kafka-consumer:v3 .
```
### Step 3: Deploy image ke Registry
kali ini saya menggunakan **Quay.io** sebagai registry, karena memiliki fitur untuk security scan image kita.
PS: saya menggunakan skopeo untuk mempermudah perpindahan registry
```bash
$ skopeo --insecure-policy copy --dest-creds=$CREDENTIAL docker-daemon:quarkus/kafka-consumer:v3 docker://quay.io/efeluzy/quarkus-kafka-consumer:v3
```
untuk credential pada Quay.io dapat di atur di konfigurasi security pada Quay.io

### Step 4: Deploy image ke Openshift
kali saya akan menggunakan Openshift CLI untuk mendeploy aplikasi. 
> PS: developer dapat menggunakan web console untuk cara yang lebih mudah
```bash
$ oc new-app quay.io/efeluzy/quarkus-kafka-consumer:v3 --name quarkus-kafka-consumer 
```
### Bonus! Step 4 (Optional): Deploy image as Serverless apps
```bash
$ oc login
$ oc project erfin-serverless-demo
$ kn service create quarkus-kafka-consumer --image quay.io/efeluzy/quarkus-kafka-consumer:v3
```
result:
```bash
Creating service 'quarkus-kafka-consumer' in namespace 'erfin-serverless-demo':

  0.693s The Route is still working to reflect the latest desired specification.
  0.694s Configuration "quarkus-kafka-consumer" is waiting for a Revision to become ready.
 14.813s ...
 14.991s Ingress has not yet been reconciled.
 15.277s Ready to serve.

Service 'quarkus-kafka-consumer' created to latest revision 'quarkus-kafka-consumer-gplbc-1' is available at URL:
http://quarkus-kafka-consumer-erfin-serverless-demo.apps.erfin-cluster.sandbox1459.opentlc.com
```

Cek Serverless service menggunakan knative cli
```bash
$ kn service list
```
