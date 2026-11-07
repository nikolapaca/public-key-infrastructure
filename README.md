# PKI sistem – backend

Projekat iz predmeta **Bezbednost u sistemima elektronskog poslovanja** (2025).

PKI (Public Key Infrastructure) je sistem za upravljanje digitalnim sertifikatima koji omogućava sigurnu autentifikaciju korisnika i uređaja, kao i zaštitu komunikacije. Podržava izdavanje, pregled i povlačenje sertifikata, upravljanje korisnicima i šablonima. Drugi deo sistema je deljeni menadžer lozinki zasnovan na javnim i privatnim ključevima korisnika iz PKI sistema.

Ovaj repozitorijum sadrži **serverski deo** aplikacije. Frontend se nalazi u repozitorijumu
[public-key-infrastructure-fe](https://github.com/MilosNesh/public-key-infrastructure-fe).

## Tehnologije

- Java 11, Spring Boot 2.7, Spring Security
- JPA/Hibernate, PostgreSQL
- BouncyCastle (X.509 sertifikati)
- JWT autentifikacija, reCAPTCHA
- AES-256-GCM (zaštita lozinki keystore-ova)

## Pokretanje

1. Pokrenuti PostgreSQL i napraviti bazu `pki` (`localhost:5432`).
2. Postaviti promenljivu okruženja `APP_MASTER_SECRET` (master ključ za enkripciju).
3. Pokrenuti aplikaciju:
   ```
   ./mvnw spring-boot:run
   ```
4. Server radi na `https://localhost:8084`.

## Moj doprinos

Samostalno sam implementirao sledeće funkcionalnosti (grane `feat-HTTPS`, `feat/track-active-tokens`
i `feat-ENCRYPT-PASSWORD`, spojene u `development`).

### 1. HTTPS komunikacija

- Server radi isključivo preko HTTPS-a (`server.ssl.enabled=true`, PKCS12 keystore `ssl/keystore.p12`).
- Generisan je sopstveni CA (`myCA`) i server sertifikat potpisan tim CA-om (OpenSSL konfiguracija u `ssl/openssl.cnf`), uz `truststore.p12`.
- Dodata je CORS konfiguracija (`CorsConfig`) kako bi frontend mogao bezbedno da komunicira sa HTTPS serverom.

### 2. Praćenje aktivnih tokena

- Pri prijavi se generiše JWT koji sadrži identifikator sesije (`sid` claim). Tokeni se **ne čuvaju u bazi**,
  već se sesija identifikuje preko `sid` polja.
- `SessionInfo` i `SessionStore` (`InMemorySessionStore`) čuvaju metapodatke sesije: IP adresu, User-Agent
  (tip uređaja i pregledača), vreme poslednje aktivnosti i status opozvanosti.
- `ClientInfo` izvlači IP adresu i podatke o uređaju iz zahteva.
- `TokenAuthenticationFilter` pri svakom zahtevu proverava da li je sesija opozvana i ažurira vreme poslednje aktivnosti.
- Endpointi (`SessionController`, `/api/sessions`):
  - `GET /api/sessions` – lista aktivnih sesija korisnika,
  - `DELETE /api/sessions/{sid}` – opoziv jedne sesije (trenutna odjava sa tog uređaja),
  - `POST /api/sessions/revoke-all-except-current` – odjava sa svih ostalih uređaja.

### 3. Zaštita lozinki keystore-ova CA sertifikata

- Za svakog CA korisnika generiše se nasumičan simetrični ključ (KEK, 256 bita, `SecureRandom`) – `KekServiceImpl`.
- KEK se u tabeli `ca_secrets` čuva enkriptovan **master ključem** (`app.master.secret`, učitava se iz promenljive
  okruženja `APP_MASTER_SECRET`) algoritmom **AES-256-GCM** sa nasumičnim IV-jem i AAD vrednošću (`AesGcmService`).
- Tim KEK-om se enkriptuju nasumično generisane lozinke keystore-ova u kojima se čuvaju CA sertifikati
  (jedan KEK po CA korisniku), integrisano u `CertificateServiceImpl`.
