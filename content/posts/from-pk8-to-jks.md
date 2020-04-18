---
title: "Android : Créer un fichier keystore (jks) à partir d’une clé (.pk8) et d’un certificat (.pem)"
date: 2016-11-14T15:03:19+02:00
draft: false
---

- Extraire la clé contenue dans le fichier pk8 et la mettre en clair dans un fichier pem

```sh
openssl pkcs8 -inform DER -nocrypt -in key.pk8 -out key.pem
```

- Regrouper la clé et le certificat dans un fichier p12 temporaire

```sh
openssl pkcs12 -export -in certificate.pem -inkey key.pem -out platform.p12 -password pass:android -name AndroidDebugKey
```

- Générer le fichier jks à partir du p12 (keytool est un outil du framework android)

```sh
keytool -importkeystore -deststorepass android -destkeystore platform.jks -srckeystore platform.p12 -srcstoretype PKCS12 -srcstorepass android
```

- Vérifier le fichier

```sh
keytool -list -v -keystore platform.jks
```
