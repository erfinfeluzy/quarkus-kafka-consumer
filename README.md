# Tutorial Quarkus.io : Stream Kafka Topic ke Html dengan Server Sent Event (SSE) - Bahasa Indonesia

Tutorial ini akan melakukan hands on mengenai cara stream Kafka ke HTML dengan menggunakan metode Server Send Event (SSE).
Kelebihan SSE dibandingkan dengan menggunakan Web Socket (ws://) adalah SSE menggunakan protokol http(s) dan satu arah, hanya dari server ke client saja.

[Quarkus.io](quarkus.io), Supersonic Subatomic Java, adalah framework Kubernetes-Native Java yang dirancang khusus untuk Java Virtual Machine (JVM) seperti GraalVM dan HotSpot. Disponsori oleh [Red Hat](redhat.com), Quarkus mengoptimalkan Java secara khusus untuk Kubernetes dengan mengurangi ukuran aplikasi Java, ukuran *image container*, dan jumlah memori yang diperlukan untuk menjalankan *image* tersebut. Sebagai perbandingan, waktu yang dibutuhkan Quarkus untuk startup aplikasi REST + CRUD adalah 0,042 detik dan memory yang digunakan hanya *28 MB*. [source](https://quarkus.io)

Prerequsite tutorial ini adalah:
- Java 8 
- Maven **3.6.2 keatas** -> versi dibawah ini tidak didukung
- IDE kesayangan anda (RedHat CodeReady, Eclipse, VSCode, Netbeans, InteliJ, dll)
- Git client
- Dasar pemrograman Java

## Step 1: Install Apache Kafka dan buat topik baru
```bash
> wget https://www.apache.org/dyn/closer.cgi?path=/kafka/2.4.1/kafka_2.12-2.4.1.tgz
> tar -xzf kafka_2.12-2.4.1.tgz
> cd kafka_2.12-2.4.1
```

### Jalankan Kafka Server
```bash
> bin/zookeeper-server-start.sh config/zookeeper.properties
> bin/kafka-server-start.sh config/server.properties
```
Langkah diatas dilakukan untuk menjalankan server zookeeper di port **2181** dan server Kafka di port **9092**

### Buat Topic baru
```bash
> bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic mytopic
```
Langkah diatas dilakukan untuk membuat Topic Kafka baru dengan nama *mytopic* dengan jumlah partisi *1* dengan faktor replikasi sebanyak *1*

## Step 2: Clone code dari GitHub saya
Download source code dari repository [GitHub](https://github.com/erfinfeluzy/training-quarkus-sse-kafka.git) saya:
```bash
> git clone https://github.com/erfinfeluzy/training-quarkus-sse-kafka
> cd training-quarkus-sse-kafka
> mvn quarkus:dev
```
> Note : Mode ini digunakan menggunakan mode *Development*, untuk menjalankan secara native dapat menggunakan
```
> mvn package -Pnative
```

Struktur code sebagai berikut:

![code structure](https://github.com/erfinfeluzy/training-quarkus-sse-kafka/blob/master/code-structure.png?raw=true)

Buka browser dengan url [http://localhost:8080](http://localhost:8080). akan terlihat halaman sbb:

![code structure](https://github.com/erfinfeluzy/training-spring-sse-kafka/blob/master/result-on-browser.png?raw=true)

> Note: Untuk browser modern (eg: Chrome, Safari,etc) sudah mensupport untuk Server Sent Event (SSE). Hal ini mungkin tidak berjalan di IE :)

## Pembahasan Kode

### Library yang dibutuhkan
Tambahkan library kafka client pada file pom.xml
```xml
<dependency>
	<groupId>io.quarkus</groupId>
	<artifactId>quarkus-smallrye-reactive-messaging-kafka</artifactId>
</dependency>

...

<dependency>
	<groupId>io.quarkus</groupId>
	<artifactId>quarkus-scheduler</artifactId>
</dependency>
```
> Note:
> 1. Quarkus menggunakan library dari [Smallrye.io](https://smallrye.io/) sebagai kafka client.
> 2. Komponen *Quarkus Scheduler* digunakan untuk mensimulasikan traffic ke Kafka Server sebagai topic publisher.

### Konfigurasi Kafka Consumer
```properties
# Configure the Kafka source (we read from it)
mp.messaging.incoming.mytopic-subscriber.connector=smallrye-kafka
mp.messaging.incoming.mytopic-subscriber.topic=mytopic
mp.messaging.incoming.mytopic-subscriber.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```
### Konfigurasi Kafka Producer
```properties
mp.messaging.outgoing.mytopic-publisher.connector=smallrye-kafka
mp.messaging.outgoing.mytopic-publisher.topic=mytopic
mp.messaging.outgoing.mytopic-publisher.value.serializer=org.apache.kafka.common.serialization.StringSerializer
```

### Publish Random message ke Kafka Server setiap 5 detik

Snippet berikut pada file *KafkaTopicGenerator.java* untuk mempublish random data ke Topic dengan nama **mytopic**.
```java
	@Inject
	@Channel("mytopic-publisher")
	private Emitter<String> emitter;

	@Scheduled(every = "5s")
	public void scheduler() {
		
		//randomly generate kafka message to topic:mytopic every 5 seconds	
		emitter.send( "Data tanggal : " + new Date () + "; id : " + UUID.randomUUID() );
	}
```
> Note:
> 1. **@Channel("mytopic-publisher")** adalah channel stream internal microprofile. *mytopic-pulisher* dikonfigurasikan pada application.properties untuk diteruskan ke Kafka Server dengan nama topic : *mytopic*.
> 2. **@Scheduled(every = "5s")** digunakan untuk mensimulasikan traffic data masuk ke Kafka Server setiap 5 detik.

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
	const eventSource = new EventSource('http://localhost:8080/stream');
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
Coba di browser pada url berikut [http://localhost:8080](http://localhost:8080).

## Bonus! Deploy aplikasi kamu ke Red Hat Openshift
Deploy aplikasi kamu secara native dan secure diatas [Red Hat OpenShift](https://www.openshift.com/) dan [Red Hat Secured Registry Quay.io](https://quay.io) pada tutorial saya [https://github.com/erfinfeluzy/quarkus-demo](https://github.com/erfinfeluzy/quarkus-demo).
