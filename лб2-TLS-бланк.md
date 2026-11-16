<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ |Іванюша Ярослав Ігорович |
| Група |Кб 2.02 |
| Ідентифікатор |cs-202-09|
| Дата виконання |15.11.2026 |
| Середовище |GitHub Codespaces |
| Версія OpenSSL |OpenSSL 3.0.13 30 Jan 2024 |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | `gate.cs-202-09.lab` |
| Імена для сертифіката |`www.cs-202-09.lab`, `app.cs-202-09.lab` |
| Ваш сервер (IP) |`10.53.1.1` |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```
root@cs-202-09:/lab/work# timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt &
[1] 43
root@cs-202-09:/lab/work# sleep 1; curl -s http://gate.cs-202-09.lab/; wait
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: gate.cs-202-09.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
root@cs-202-09:/lab/work# echo "--- GREP ---"
 GREP 
root@cs-202-09:/lab/work# grep -a -E 'GET /|Host:|server: web' t1-http.txt
19:14:04.000549 IP 10.53.1.1.39828 > 10.53.0.80.80: Flags [P.], seq 1:82, ack 1, win 502, options [nop,nop,TS val 671629963 ecr 806240897], length 81: HTTP: GET / HTTP/1.1
(.B.0.B.GET / HTTP/1.1
Host: gate.cs-202-09.lab
server: web
Host: gate.cs-202-09.lab
```

Вивід кроку 1.2:

```
root@cs-202-09:/lab/work# cd /lab/work
timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt &
sleep 1; curl -s --cacert $STAND_CA https://gate.cs-202-09.lab/; wait
echo "--- COUNT1 ---"
grep -a -c -E 'GET /|server: web' t1-https.txt
echo "--- COUNT2 ---"
grep -a -o 'gate.cs-202-09.lab' t1-https.txt | wc -l
echo "--- END ---"
[1] 59
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: gate.cs-202-09.lab
SNI: gate.cs-202-09.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
 COUNT1 
0
 COUNT2 
1

```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` |GET / HTTP/1.1 |0 |
| заголовок `Host` |gate.cs-202-09.lab |- |
| тіло відповіді (`server: web`) |server: web' t1-https.txt |0 |
| ім'я віддаленого сервера (кількість входжень) |0 |1 |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?
 Ім'я сервера передається в розширенні server_name (скорочено SNI — Server Name Indication) повідомлення ClientHello протоколу TLS. ClientHello — це перше повідомлення рукостискання, яке клієнт надсилає серверу. Розширення SNI входить до складу ClientHello і містить поле host_name з ім'ям, до якого звертається клієнт (gate.cs-202-09.lab). У дампі tcpdump це ім'я видно відкритим текстом, оскільки SNI передається до встановлення шифрування.

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?
SNI - це розширення протоколу TLS, а не HTTP. У разі звернення за протоколом HTTP (порт 80) рукостискання TLS не виконується взагалі, тому клієнт не передає SNI, сервер його не отримує і у відповіді виводить поле SNI: без значення. У разі HTTPS (порт 443) клієнт спочатку виконує повне рукостискання TLS, у ClientHello передає server_name=gate.cs-202-09.lab, і сервер, отримавши SNI, відображає його в заголовку відповіді (

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?
Версію HTTP у другому випадку визначило узгодження через розширення ALPN. curl за замовчуванням пропонує в ClientHello перелік підтримуваних прикладних протоколів - зазвичай h2 і http/1.1. Сервер вибирає один із запропонованих, у цьому разі h2, і повертає його у розширенні ALPN повідомлення ServerHello. Після цього весь HTTP-обмін ведеться версією 2.0. У разі HTTP-запиту протоколу TLS немає, ALPN не застосовується, тому використовується версія за замовчуванням - HTTP/1.1.

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```
ALPN / VERIFY
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)
 RECORDS 
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=329
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=985
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=75
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
```

Вивід кроку 2.2 (TLS 1.2):

```
ALPN / VERIFY 
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)
 RECORDS
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=224
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=980
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=111
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=214
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)
```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 |К→С |22 (Handshake) |- |ClientHello |329 |
|2|С→К|22 |-|ServerHello|118|
|3|С→К|20 (ChangeCipherSpec)|-|ChangeCipherSpec	|-|
|4|С→К|23 (ApplicationData)|22|EncryptedExtensions|15|
|5|С→К|23|22|Certificate|985|
|6|С→К|23|22|CertificateVerify|75|
|7|С→К|23|22|Finished|48|
|8|К→С|20|-|ChangeCipherSpec|-|
|9|К→С|23|22|Finished|48|
|10|К→С|23|21 (Alert)|Alert (close_notify)|-|


Вивід кроків 2.3 і 2.4:

```
        extension_type=server_name(0), length=23
          0000 - 00 15 00 00 12 67 61 74-65 2e 63 73 2d 32 30   .....gate.cs-20
          000f - 32 2d 30 39 2e 6c 61 62-                       2-09.lab
        extension_type=ec_point_formats(11), length=4

root@cs-202-09:/lab/work# cd /lab/work
for v in 1.3 1.2; do
  timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt &
  sleep 1; curl -s -o /dev/null --tlsv1.2 --tls-max $v --cacert $STAND_CA https://gate.cs-202-09.lab/; wait
  echo "TLS $v: $(grep -a -c 'OPISM Stand Intermediate' t2-dump-$v.txt)"
done
[1] 38
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
23 packets captured
23 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 44
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2
```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт |329 |	224 |
| Номер першого запису, тип якого спостерігач не бачить |4 (EncryptedExtensions) |- |
| Кількість RTT від ClientHello до першого HTTP-запиту |1 |	2|
| Чи видно сертифікат у дампі (результат 2.4) |0 |2 |
| Обраний протокол ALPN |h2|h2|

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?
Шифрування починається з 4-го запису у ньому Content Type = ApplicationData (23), і OpenSSL додатково виводить Inner Content Type = Handshake (22) тобто справжній тип (Handshake) приховано, а назовні видно лише «ApplicationData». Починаючи з цього запису й до кінця рукостискання всі наступні записи мають Content Type = ApplicationData (23). Для кожного з них спостерігач у мережі бачить лише Content Type = 23 (ApplicationData) і довжину запису але не може визначити, що всередині. У TLS 1.2 такого немає там ChangeCipherSpec лише позначає межу, після якої вміст записів стає зашифрованим, але Content Type усе одно видно (22/20/23), бо в TLS 1.2 заголовок запису не шифрується.

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.
ClientHello у TLS 1.3 містить додаткові розширення, яких немає в 1.2: key_share, supported_versions , psk_key_exchange_modes, signature_algorithms_cert, post_handshake_auth та інші. Саме розширення key_share дозволяє клієнту надіслати свою частину ECDHE-обміну вже в ClientHello сервер одразу в ServerHello відповідає своєю частиною і похідним ключем, і рукостискання завершується за 1 RTT замість 2 RTT у 1.2, де для ServerKeyExchange/ClientKeyExchange потрібен окремий оберт.

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?
Розширення server_name складається з
ServerNameList length — 2 байти;
NameType — 1 байт (0 = host_name);
HostName length — 2 байти;
власне HostName — N байт (ASCII).
Формула: довжина = N + 5.

Для імені з 12 символів: 12 + 5 = 17 байт.
Для мого імені gate.cs-202-09.lab (N = 18): 18 + 5 = 23 байти.

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?
ChangeCipherSpec у TLS 1.3 передається виключно для сумісності з проміжними пристроями, які розпізнають TLS за наявністю CCS у потоці. У TLS 1.3 він нічого не змінює: шифрування вмикається не за CCS, а за фактом переходу до зашифрованих записів після ServerHello. Тому CCS у 1.3 має вигляд фіктивного запису, який OpenSSL позначає окремо (Content Type = ChangeCipherSpec (20)), але ігнорує його вміст. Якби CCS прибрали, старі middlebox'и могли б обірвати з'єднання, вважаючи його не-TLS.

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 0bd63a
issuer=CN = Pebble Root CA 0bd63a
notBefore=Nov 16 20:37:48 2026 GMT
notAfter=Nov 16 20:37:48 2056 GMT
 3.1 CERT FILES 
-rw-rw-rw- 1 root root 1151 Nov 16 20:38 acme-root.crt
-rw-rw-rw- 1 root root  623 Nov 16 19:05 server.crt
-rw------- 1 root root  241 Nov 16 19:05 server.key
 3.2 CURL BEFORE 
curl: (60) SSL certificate problem: self-signed certificate
... (решта тексту curl)
curl_exit=60
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```
2026/11/16 20:49:08 No key found for account admin@cs-202-09.lab. Generating a P256 key.
2026/11/16 20:49:08 Saved key to /lab/work/lego/accounts/ca.lab_14000/admin@cs-202-09.lab/keys/admin@cs-202-09.lab.key
2026/11/16 20:49:09 [INFO] acme: Registering account for admin@cs-202-09.lab
!!!! HEADS UP !!!!

Your account credentials have been saved in your Let's Encrypt
configuration directory at "/lab/work/lego/accounts".

You should make a secure backup of this folder now. This
configuration directory will also contain certificates and
private keys obtained from Let's Encrypt so making regular
backups of this folder is ideal.
2026/11/16 20:49:09 [INFO] [www.cs-202-09.lab, app.cs-202-09.lab] acme: Obtaining SAN certificate
2026/11/16 20:49:09 [INFO] [app.cs-202-09.lab] AuthURL: https://ca.lab:14000/authZ/Pob99vao09y49AehSKNbfI3sd8xuTEuiP5UpQEkZJPc
2026/11/16 20:49:09 [INFO] [www.cs-202-09.lab] AuthURL: https://ca.lab:14000/authZ/rJeSjsZwILRwf2KEhg4KmQeKxbGGy58I9_T12iVgBf0
2026/11/16 20:49:09 [INFO] [www.cs-202-09.lab] acme: Could not find solver for: tls-alpn-01
2026/11/16 20:49:09 [INFO] [www.cs-202-09.lab] acme: use http-01 solver
2026/11/16 20:49:09 [INFO] [app.cs-202-09.lab] acme: Could not find solver for: tls-alpn-01
2026/11/16 20:49:09 [INFO] [app.cs-202-09.lab] acme: use http-01 solver
2026/11/16 20:49:09 [INFO] [www.cs-202-09.lab] acme: Trying to solve HTTP-01
2026/11/16 20:49:16 [INFO] [www.cs-202-09.lab] The server validated our request
2026/11/16 20:49:16 [INFO] [app.cs-202-09.lab] acme: Trying to solve HTTP-01
2026/11/16 20:49:20 [INFO] [app.cs-202-09.lab] The server validated our request
2026/11/16 20:49:20 [INFO] [www.cs-202-09.lab, app.cs-202-09.lab] acme: Validations succeeded; requesting certificates
2026/11/16 20:49:20 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/16 20:49:20 [INFO] [www.cs-202-09.lab] Server responded with a certificate.
NGINX ACME LOG 
10.53.0.10 - - [16/Nov/2026:20:49:09 +0000] "GET /.well-known/acme-challenge/UCSx4QJuGQzlShy4mILknhUBvyaorAVyZQgZOZDyZQg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:20:49:09 +0000] "GET /.well-known/acme-challenge/UCSx4QJuGQzlShy4mILknhUBvyaorAVyZQgZOZDyZQg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:20:49:09 +0000] "GET /.well-known/acme-challenge/UCSx4QJuGQzlShy4mILknhUBvyaorAVyZQgZOZDyZQg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:20:49:16 +0000] "GET /.well-known/acme-challenge/K1ZQVlBh9MhMg70HtQTc2gSfnzo6c0uovj_xCIFzuBM HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:20:49:16 +0000] "GET /.well-known/acme-challenge/K1ZQVlBh9MhMg70HtQTc2gSfnzo6c0uovj_xCIFzuBM HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:20:49:16 +0000] "GET /.well-known/acme-challenge/K1ZQVlBh9MhMg70HtQTc2gSfnzo6c0uovj_xCIFzuBM HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
 FILES 
total 16
-rw------- 1 root root  989 Nov 16 20:49 www.cs-202-09.lab.crt
-rw------- 1 root root 1208 Nov 16 20:49 www.cs-202-09.lab.issuer.crt
-rw------- 1 root root  158 Nov 16 20:49 www.cs-202-09.lab.json
-rw------- 1 root root  227 Nov 16 20:49 www.cs-202-09.lab.key
```

Вивід кроку 3.4:

```
SERVER CERT 
subject=CN = www.cs-202-09.lab
issuer=CN = Pebble Intermediate CA 034b53
notBefore=Nov 16 20:49:20 2026 GMT
notAfter=Nov 16 20:49:19 2031 GMT
serial=78CF418EF99E856A
X509v3 Subject Alternative Name: 
    DNS:www.cs-202-09.lab, DNS:app.cs-202-09.lab
 ISSUER CERT 
subject=CN = Pebble Intermediate CA 034b53
issuer=CN = Pebble Root CA 0bd63a
notBefore=Nov 16 20:37:48 2026 GMT
notAfter=Nov 16 20:37:48 2056 GMT
```

Вивід кроку 3.5:

```
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/16 20:54:34 [notice] 49#49: signal process started
 CURL 3.5 
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
curl_exit=60
 S_CLIENT 3.5 
Certificate chain
 0 s:CN = www.cs-202-09.lab
   i:CN = Pebble Intermediate CA 034b53
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 20:49:20 2026 GMT; NotAfter: Nov 16 20:49:19 2031 GMT
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/16 20:56:37 [notice] 57#57: signal process started
 CURL 3.6 WWW 
server: cs-202-09 (https)
curl_exit=0
 CURL 3.6 APP 
server: cs-202-09 (https)
curl_exit=0
 S_CLIENT 3.6 
Certificate chain
 0 s:CN = www.cs-202-09.lab
   i:CN = Pebble Intermediate CA 034b53
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 20:49:20 2026 GMT; NotAfter: Nov 16 20:49:19 2031 GMT
 1 s:CN = Pebble Intermediate CA 034b53
   i:CN = Pebble Root CA 0bd63a
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 20:37:48 2026 GMT; NotAfter: Nov 16 20:37:48 2056 GMT

Verify return code: 0 (ok)
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера |CN = Pebble Intermediate CA 034b53 |
| Видавець сертифіката видавця |CN = Pebble Root CA 0bd63a |
| Імена в SAN |DNS:www.cs-202-09.lab, DNS:app.cs-202-09.lab |
| Строк дії (від – до) |Nov 16 20:49:20 2026 GMT – Nov 16 20:49:19 2031 GMT |
| Тип ключа сервера і алгоритм підпису |PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256 |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 |3.5 = 1, 3.6 = 2 |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.
Контроль перевірив центр сертифікації Pebble за допомогою HTTP-01 challenge протоколу ACME. Механізм такий: lego згенерував для кожного імені окремий токен, розмістив файл /.well-known/acme-challenge/<token> у webroot /var/www/acme, і попросив Pebble перевірити. Pebble (з адреси 10.53.0.10, user-agent LetsEncrypt-Pebble-VA) звернувся до http://www.cs-202-09.lab/.well-known/acme-challenge/... і http://app.cs-202-09.lab/.well-known/acme-challenge/... — і nginx віддав 200.

[INFO] [www.cs-202-09.lab] acme: Trying to solve HTTP-01
[INFO] [www.cs-202-09.lab] The server validated our request
[INFO] [app.cs-202-09.lab] acme: Trying to solve HTTP-01
[INFO] [app.cs-202-09.lab] The server validated our request

"GET /.well-known/acme-challenge/UCSx4QJuGQzlShy4mILknhUBvyaorAVyZQgZOZDyZQg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA"
"GET /.well-known/acme-challenge/K1ZQVlBh9MhMg70HtQTc2gSfnzo6c0uovj_xCIFzuBM HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA"

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?
Тому що в кроці 3.5 у /lab/tls/server.crt лежав лише сертифікат сервера — без проміжного сертифіката Pebble Intermediate CA 034b53. Клієнт отримав від сервера ланцюжок з одного сертифіката; curl бачить, що issuer = Pebble Intermediate CA 034b53, але цього проміжного немає ні в отриманому ланцюжку, ні в --cacert. Тому curl не може побудувати ланцюжок «сервер → проміжний → корінь» і повертає помилку unable to get local issuer certificate (код 60). s_client те саме показує кодом 21 (unable to verify the first certificate) і ланцюжком з одним сертифікатом.

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?
id-ecPublicKey, 256 (bit) — це тип публічного ключа самого сервера: еліптична крива (ECDSA/ECDHE) довжиною 256 біт (P-256). Цей ключ сервер використовує в TLS для обміну ключами (ECDHE) і, за потреби, для підпису в CertificateVerify.

RSA-SHA256 — це алгоритм підпису сертифіката: хто і чим підписав цей сертифікат. У нашому разі серверний сертифікат підписав проміжний CA Pebble Intermediate CA 034b53 своїм RSA-ключем 2048 біт з хешем SHA-256 (це видно в сертифікаті проміжного: PKEY: rsaEncryption, 2048 (bit)).

Ці значення не суперечать, бо описують різні об'єкти: id-ecPublicKey — це ключ суб'єкта (сервера), а RSA-SHA256 — це підпис видавця (проміжного CA). У X.509 це нормальна ситуація: суб'єкт може мати EC-ключ, а видавець — RSA-ключ, і навпаки.

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?
Після перезапуску стенда (або codespace) корінь Pebble генерується заново у моєму випадку Pebble Root CA 0bd63a зміниться на інший випадковий суфікс.
старий acme-root.crt більше не відповідає новому кореню;
сертифікат сервера, виданий у кроці 3.3, підписаний старим проміжним, який веде до старого кореня, тому curl --cacert /lab/tls/acme-root.crt https://www.cs-202-09.lab/ знову впаде з unable to get local issuer certificate;
щоб відновити, треба повторити кроки 3.1, 3.3  і 3.6.
У реальній інфраструктурі це відповідає ситуації, коли CA перевипустив свій корінь


## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```
 f1.cs-202-09.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f1.cs-202-09.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Apr 30 19:05:19 2026 GMT; NotAfter: Oct 27 19:05:19 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 16 19:05:19 2026 GMT; NotAfter: Nov 15 19:05:19 2031 GMT

Verify return code: 10 (certificate has expired)
 f2.cs-202-09.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f2.cs-202-09.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 16 19:05:19 2026 GMT; NotAfter: Feb 14 19:05:19 2027 GMT

Verify return code: 21 (unable to verify the first certificate)
 f3.cs-202-09.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f3.cs-202-09.lab'
Certificate chain
 0 s:CN = old.cs-202-09.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 16 19:05:20 2026 GMT; NotAfter: Feb 14 19:05:20 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 16 19:05:19 2026 GMT; NotAfter: Nov 15 19:05:19 2031 GMT

Verify return code: 0 (ok)
 f4.cs-202-09.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f4.cs-202-09.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   v:NotBefore: Nov 16 19:05:20 2026 GMT; NotAfter: Feb 14 19:05:20 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   v:NotBefore: Nov 16 19:05:19 2026 GMT; NotAfter: Nov 15 19:05:19 2031 GMT

Verify return code: 20 (unable to get local issuer certificate)
 f5.cs-202-09.lab
server: web, f5
Certificate chain
 0 s:CN = f5.cs-202-09.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 16 19:05:20 2026 GMT; NotAfter: Feb 14 19:05:20 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 16 19:05:19 2026 GMT; NotAfter: Nov 15 19:05:19 2031 GMT

Verify return code: 0 (ok)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 |certificate has expired |	10 (certificate has expired) |Перевірка строку дії |v:NotBefore: Apr 30 19:05:19 2026 GMT; NotAfter: Oct 27 19:05:19 2026 GMT |Перевипустити сертифікат з актуальним строком дії |
| f2 |unable to get local issuer certificate |21 (unable to verify the first certificate) |бракує проміжного сертифіката |У Certificate chain лише 0 s:CN = f2..., i:OPISM Stand Intermediate CA; запису 1 s:... Intermediate немає |Додати проміжний сертифікат у файл, який віддає сервер |
| f3 |no alternative certificate subject name matches target host name 'f3.cs-202-09.lab' |0 (ok) |Перевірка відповідності імені хоста (hostname) до SAN |curl: ... no alternative certificate subject name matches ...; у ланцюжку s:CN = old.cs-202-09.lab |Випустити сертифікат, у SAN якого є f3.cs-202-09.lab |
| f4 |unable to get local issuer certificate |20 (unable to get local issuer certificate) | ланцюжок веде до CA, якого немає в $STAND_CA |Ланцюжок повний: 1 s:O = Foreign Org, CN = Foreign Intermediate CA, i:O = Foreign Org, CN = Foreign Root CA |Випустити сертифікат у CA, чий корінь є в довірених у клієнта |
| f5 |- |0 (ok) |- |server: web, f5; Verify return code: 0 (ok) |- |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | | | |
| `time_appconnect` | | | |
| `time_starttransfer` | | | |

Вивід кроків 5.2 і 5.3:

```text

```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | | | | | | |
| TLS 1.2, нове | | | | | | |
| 5.3, перший запит | | | | | | |
| 5.3, другий запит | | | | | | |
| 5.3, третій запит | | | | | | |

RTT, визначений вимірюванням, і з якого значення його визначено:

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

## Декларація використання ШІ

Використовувалися інструменти ШІ: <!-- так / ні -->

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | | | |
