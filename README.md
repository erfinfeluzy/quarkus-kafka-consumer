# Tutorial Quarkus.io : Consume Kafka Topic and Stream as API using Server Sent Event (SSE) - Bahasa Indonesia

Tutorial ini akan melakukan hands on mengenai cara stream Kafka ke HTML dengan menggunakan metode Server Send Event (SSE).
Kelebihan SSE dibandingkan dengan menggunakan Web Socket (ws://) adalah SSE menggunakan protokol http(s) dan satu arah, hanya dari server ke client saja.


Prerequsite tutorial ini adalah:
- Java 8 
- Maven **3.6.2 keatas** -> versi dibawah ini tidak didukung
- IDE kesayangan anda (RedHat CodeReady, Eclipse, VSCode, Netbeans, InteliJ, dll)
- Git client
- Dasar pemrograman Java
- Untuk build native image diperlukan Docker runtime
- Pre-installed Kafka cluster on Openshift 


## Step 1: Clone code dari GitHub saya
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
mp.messaging.incoming.mytopic-subscriber.connector=smallrye-kafka
mp.messaging.incoming.mytopic-subscriber.topic=mytopic
mp.messaging.incoming.mytopic-subscriber.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```


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
Coba di browser pada url berikut [http://localhost:8080](http://localhost:8080).

## Bonus! Deploy aplikasi kamu ke Red Hat Openshift
Deploy aplikasi kamu secara native dan secure diatas [Red Hat OpenShift](https://www.openshift.com/) dan [Red Hat Secured Registry Quay.io](https://quay.io) pada tutorial saya [https://github.com/erfinfeluzy/quarkus-demo](https://github.com/erfinfeluzy/quarkus-demo).
