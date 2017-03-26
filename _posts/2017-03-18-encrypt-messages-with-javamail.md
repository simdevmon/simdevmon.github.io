---
title: Encrypt messages with JavaMail
updated: 2017-03-18 18:00
---

I want to process an inbox with encrypted messages automatically by using JavaMail. 
To encrypt the messages there is a **certificate.p12** with a **passphrase** available.  
**BouncyCastle** is used for message encryption in this example.

For the JavaMail connection and certificate I use the following attributes:
```java
    String host;
    int port;
    String protocol;
    String username;
    String password;
    String folder;

    String certFilename;
    String certPassword;
```

As the first step I extract the **PrivateKey** and **X509Certificate**, which is needed to encrypt messages. 
```java
// Extract private key and certificate
KeyStore ks = KeyStore.getInstance("PKCS12");
ks.load(new FileInputStream(certFilename), certPassword.toCharArray());
String keyAlias = getKeyAlias(ks);
PrivateKey privateKey = (PrivateKey) ks.getKey(keyAlias, certPassword.toCharArray());
X509Certificate cert = (X509Certificate) ks.getCertificate(keyAlias);
...
        
private String getKeyAlias(KeyStore ks) throws KeyStoreException {
    Enumeration aliases = ks.aliases();
    while (aliases.hasMoreElements()) {
        String alias = (String) aliases.nextElement();
        if (ks.isKeyEntry(alias)) {
            return alias;
        }
    }
    throw new IllegalStateException("Cannot find private key.");
}
```

Now I can process all messages and encrypt them:
```java
...
// Process emails from inbox
Store store = getConnectedStore();
Folder inbox = store.getFolder(folder);
inbox.open(Folder.READ_WRITE);
Message[] msgs = inbox.getMessages();
for (Message msg : msgs) {
    if (isEncrypted(msg)) {
        processMessage(encrypt(privateKey, cert, (MimeMessage) msg));
    }
}

private Store getConnectedStore() throws MessagingException
{
    Properties props = new Properties();
    props.put("mail.smtp.host", host);
    props.put("mail.protocol.port", port);
    Store store = Session.getInstance(props).getStore(protocol);
    store.connect(host, username, password);
    if (!store.isConnected())
    {
        throw new IllegalStateException("Cannot connect to store.");
    }
    return store;
}

private boolean isEncrypted(Message msg) throws MessagingException {
    return msg.getContentType().contains("application/pkcs7-mime")
        && msg.getContentType().contains("smime-type=enveloped-data");
}

private MimeBodyPart encrypt(PrivateKey privateKey, X509Certificate cert, MimeMessage msg)
    throws IOException, MessagingException, CMSException, NoSuchProviderException, SMIMEException {
    RecipientId recId = new RecipientId();
    recId.setSerialNumber(cert.getSerialNumber());
    recId.setIssuer(cert.getIssuerX500Principal().getEncoded());
    return SMIMEUtil.toMimeBodyPart(
        new SMIMEEnveloped(msg).getRecipientInfos().get(recId).getContent(privateKey, "SunJCE")
    );
}

private void processMessage(MimeBodyPart encrypted) throws IOException, MessagingException {
    if (encrypted.getContent() instanceof Multipart) {
        Multipart multi = (Multipart) encrypted.getContent();
        // ...
    }
}
```

This is just a simple example to show that encryption can be done quite easily.