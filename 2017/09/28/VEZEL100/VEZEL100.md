---
title: 0CTF 2015 Quals CTF VEZEL
date: 2017-09-28 16:49:43
tags: 
 - android
---

# 

**Category:** Mobile
**Points:** 100
**Solves:** 105
**Description:** 

> Evermars says he is good at repackaging Android applications, for [example](vezel.apk).

## Write-up

分析过程可以参考下面三个链接, 程序很简单flag由两个字符串相加内容计算, 其中一个getCRC可以直接得到, 而apk签名可以用Main.class运行得到, 比较笨的办法是自己写一个apk, 利用PackageManager得到签名.

## Other write-ups and resources

* <http://capturetheswag.blogspot.com.au/2015/03/0ops-ctf-qualifiers-2015-vezel-mobile.html>
* <http://dakutenpura.hatenablog.com/entry/2015/03/31/002551>
* <http://www.purpleroc.com/MD/0ctf_WriteUp_vezel.html>


```Java
import java.io.IOException;
import java.io.InputStream;
import java.lang.ref.WeakReference;
import java.security.Signature;
import java.security.cert.*;
import java.util.Enumeration;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;
import java.util.logging.Level;
import java.util.logging.Logger;
 
public class Main {
 
 private static final Object mSync = new Object();
 private static WeakReference<byte[]> mReadBuffer;
 
 public static void main(String[] args) {
  if (args.length < 1) {
   System.out.println("Usage: java -jar GetAndroidSig.jar <apk/jar>");
   System.exit(-1);
  }
 
  System.out.println(args[0]);
 
  String mArchiveSourcePath = args[0];
 
  WeakReference<byte[]> readBufferRef;
  byte[] readBuffer = null;
  synchronized (mSync) {
   readBufferRef = mReadBuffer;
   if (readBufferRef != null) {
    mReadBuffer = null;
    readBuffer = readBufferRef.get();
   }
   if (readBuffer == null) {
    readBuffer = new byte[8192];
    readBufferRef = new WeakReference<byte[]>(readBuffer);
   }
  }
 
  try {
   JarFile jarFile = new JarFile(mArchiveSourcePath);
   java.security.cert.Certificate[] certs = null;
 
   Enumeration entries = jarFile.entries();
   while (entries.hasMoreElements()) {
    JarEntry je = (JarEntry) entries.nextElement();
    if (je.isDirectory()) {
     continue;
    }
    if (je.getName().startsWith("META-INF/")) {
     continue;
    }
    java.security.cert.Certificate[] localCerts = loadCertificates(jarFile, je, readBuffer);
    if (false) {
     System.out.println("File " + mArchiveSourcePath + " entry " + je.getName()
         + ": certs=" + certs + " ("
         + (certs != null ? certs.length : 0) + ")");
    }
    if (localCerts == null) {
     System.err.println("Package has no certificates at entry "
         + je.getName() + "; ignoring!");
     jarFile.close();
     return;
    } else if (certs == null) {
     certs = localCerts;
    } else {
     // Ensure all certificates match.
     for (int i = 0; i < certs.length; i++) {
      boolean found = false;
      for (int j = 0; j < localCerts.length; j++) {
       if (certs[i] != null
           && certs[i].equals(localCerts[j])) {
        found = true;
        break;
       }
      }
      if (!found || certs.length != localCerts.length) {
       System.err.println("Package has mismatched certificates at entry "
           + je.getName() + "; ignoring!");
       jarFile.close();
       return; // false
      }
     }
    }
   }
 
   jarFile.close();
 
   synchronized (mSync) {
    mReadBuffer = readBufferRef;
   }
 
   if (certs != null && certs.length > 0) {
    final int N = certs.length;
     
    for (int i = 0; i < N; i++) {
     String charSig = new String(toChars(certs[i].getEncoded()));
     System.out.println("Cert#: " + i + "  Type:" + certs[i].getType()
      + "\nPublic key: " + certs[i].getPublicKey()
      + "\nHash code: " + certs[i].hashCode()
       + " / 0x" + Integer.toHexString(certs[i].hashCode())
      + "\nTo char: " + charSig);
    }
   } else {
    System.err.println("Package has no certificates; ignoring!");
    return;
   }
  } catch (CertificateEncodingException ex) {
   Logger.getLogger(Main.class.getName()).log(Level.SEVERE, null, ex);
  } catch (IOException e) {
   System.err.println("Exception reading " + mArchiveSourcePath + "\n" + e);
   return;
  } catch (RuntimeException e) {
   System.err.println("Exception reading " + mArchiveSourcePath + "\n" + e);
   return;
  }
 }
 
 private static char[] toChars(byte[] mSignature) {
    byte[] sig = mSignature;
    final int N = sig.length;
    final int N2 = N*2;
    char[] text = new char[N2];
 
    for (int j=0; j<N; j++) {
      byte v = sig[j];
      int d = (v>>4)&0xf;
      text[j*2] = (char)(d >= 10 ? ('a' + d - 10) : ('0' + d));
      d = v&0xf;
      text[j*2+1] = (char)(d >= 10 ? ('a' + d - 10) : ('0' + d));
    }
 
    return text;
    }
 
 private static java.security.cert.Certificate[] loadCertificates(JarFile jarFile, JarEntry je, byte[] readBuffer) {
  try {
   // We must read the stream for the JarEntry to retrieve
   // its certificates.
   InputStream is = jarFile.getInputStream(je);
   while (is.read(readBuffer, 0, readBuffer.length) != -1) {
    // not using
   }
   is.close();
 
   return (java.security.cert.Certificate[]) (je != null ? je.getCertificates() : null);
  } catch (IOException e) {
   System.err.println("Exception reading " + je.getName() + " in "
       + jarFile.getName() + ": " + e);
  }
  return null;
 }
}
```