# Tap to Pay on Android integration guide

Tap to Pay on Android allows merchants to accept contactless payments using a supported device or a partner-enabled app. Tap to Pay on Android allows the merchant's app to accept payments from contactless credit or debit cards, Google Pay, smartwatches, and smartphones with other digital wallets. No additional hardware is required to accept contactless payments through Tap to Pay on Android.

---

## Step 1: Obtain API Credentials

- Access the developer dashboard to create a workspace by using a certification or production MID and to obtain an API-Key and API-Secret [ by following the steps here](https://developer.fiserv.com/product/CommerceHub/docs/?path=docs/In-Person/Integrations/Mobile-SDK/Android-TTP.md&branch=preview).

<!-- theme: info -->
> These are required in the app. Please contact your account representative to obtain your `ppId` and `hostPort`.

---

## Step 2: Load Tap to Pay on Android SDK

Android SDK library is distributed via GitHub packages and can be downloaded by adding below Maven repository to your `settings.gradle` file:

 ```groovy
maven {
    url = uri("https://maven.pkg.github.com/Fiserv/ch-ttp-androidsdk")
    credentials {
        username = "USERNAME_PROVIDED_BY_FISERV"
        password = "PASSWORD_PROVIDED_BY_FISERV"
    }
}
 ```

Add the Android SDK dependency to your app by following these steps:

1. Open the app `build.gradle` file and add following item to the dependency:

    ```groovy
    dependencies {
        // Fiserv TTP Library
        implementation ('com.fiserv.ch:ttp-payment:1.0.2') {
            changing = true
        }
    }
    ```

2. Sync the project.
3. Build your APK and install it:

    ```groovy
    ./gradlew app:assembleCertRelease

    adb push ./app/build/outputs/apk/release/app-release.apk /data/local/tmp

    adb shell pm install -i com.android.vending /data/local/tmp/app-release.apk
    ```

---

## Step 3: APK Signing

Once you have successfully built the app with the Android SDK, the APK will need to be signed by Fiserv.

<!-- theme: info -->
> This step will also be required when switching from CERT to PROD or when upgrading the Android SDK version in your app. Changes to native app functionality do not require APK signing by Fiserv.

---

## Step 4: Configure the merchant account

Add the merchant configuration details based on the CERT or PROD environment.

```java
val myConfig : FiservTTPConfig = FiservTTPConfig(
    merchantId = "MERCHANT_ID",
    terminalId = "TERMINAL_ID",
    apiKey = "API_KEY",
    secretKey = "SECRET_KEY",
    ppId = "PPID_PROVIDED_BY_FISERV",
    hostPort = "HOST_PORT_PROVIDED_BY_FISERV",
    environment: Environment.CERT,
    currencyCode: Currency.USD)
```

---

## Step 5: Configure the card reader

Create an instance of `FiservTTPCardReader` to the view model, this is the main class the app will interact with.

```java
CoroutineScope(Dispatchers.Main).launch {
    try{
        FiservTTPCardReader.initializeSession(myConfig)
    } catch (fiservTTPCardReaderError:FiservTTPCardReaderException) {
        // TODO handle error
    }
}
```

<!-- theme: info -->
> The card reader must be re-initialized each time the app starts and/or returns to the foreground.

---

## Step 6: Submit a payment request

Submit a payment request to Commerce Hub.

<!--
type: tab
titles: Charges, Cancels, Refunds, Inquiry
-->

**Submit a Charges API request:**

<!-- theme: info -->
> Currently Tap to Pay on Android only supports USD.

```java
val amount = 10.99
val merchantOrderId = "merchantOrderId"
val merchantTransactionId = "merchantTransactionId"
val captureFlag = "true"
val createToken = "true"

val transactionDetailsRequest: TransactionDetailsRequest = TransactionDetailsRequest(
            captureFlag,
            merchantTransactionId,
            merchantOrderId,
            createToken )

CoroutineScope(Dispatchers.Main).launch {
    try{
        FiservTTPCardReader.charges(
        amount.toBigDecimal(), 
        PaymentTransactionType.CAPTURE,
        transactionDetailsRequest)
    } catch (fiservTTPCardReaderError:FiservTTPCardReaderException) {
        // TODO handle error
    }
}
```

<!--
type: tab
-->

**Submit a Cancels API request:**

At least one reference transaction identifier must be provided to perform a cancel.

<!-- theme: info -->
> To support partial cancels it must be configured in Merchant Boarding and Configuration. Please contact your account representative for details.

```java
val amount = 10.99
val referenceMerchantTransactionId = "referenceMerchantTransactionId"

val referenceTransactionDetails: ReferenceTransactionDetails = ReferenceTransactionDetails(
    referenceMerchantTransactionId )

CoroutineScope(Dispatchers.Main).launch {
    try{
        FiservTTPCardReader.cancels(
            amount.toBigDecimal(),
            referenceTransactionDetails)
    } catch (fiservTTPCardReaderError:FiservTTPCardReaderException) {
        // TODO handle error
    }
}
```

<!--
type: tab
-->

**Submit a Refund API request for a payment without tap:**

At least one reference transaction identifier must be provided to perform a tagged refund.

<!-- theme: warning -->
> When processing a PIN debit refund request, submit a refund payment with tap using an unmatched tagged refund.

```java
val amount = 10.99
val merchantOrderId = "merchantOrderId"
val merchantTransactionId = "merchantTransactionId"
val captureFlag = "true"
val createToken = "true"
val referenceMerchantTransactionId = "referenceMerchantTransactionId"

val transactionDetailsRequest: TransactionDetailsRequest = TransactionDetailsRequest(
    captureFlag,
    merchantTransactionId,
    merchantOrderId,
    createToken )

val referenceTransactionDetails: ReferenceTransactionDetails = ReferenceTransactionDetails(
    referenceMerchantTransactionId )

CoroutineScope(Dispatchers.Main).launch {
    try{
        FiservTTPCardReader.refunds(
            amount.toBigDecimal(),
            RefundTransactionType.TAGGED,
            transactionDetailsRequest,
            referenceTransactionDetails)
    } catch (fiservTTPCardReaderError:FiservTTPCardReaderException) {
        // TODO handle error
    }
}
```

**Submit a Refund API request for a payment with tap:**

The `fiservTTPCardReader.refundCard` API supports both unmatched tagged refunds and open refunds.

At least one reference transaction identifier must be provided to perform an unmatched tagged refund.

<!-- theme: warning -->
> If the card used for the refund request is a PIN debit, the user will see a PIN entry screen after tapping the card.

```java
val amount = 10.99
val merchantOrderId = "merchantOrderId"
val merchantTransactionId = "merchantTransactionId"
val captureFlag = "true"
val createToken = "true"
val referenceMerchantTransactionId = "referenceMerchantTransactionId"

val transactionDetailsRequest: TransactionDetailsRequest = TransactionDetailsRequest(
    captureFlag,
    merchantTransactionId,
    merchantOrderId,
    createToken )

val referenceTransactionDetails: ReferenceTransactionDetails = ReferenceTransactionDetails(
    referenceMerchantTransactionId )

CoroutineScope(Dispatchers.Main).launch {
    try{
        FiservTTPCardReader.refunds(
            amount.toBigDecimal(),
            RefundTransactionType.UNMATCHED,
            transactionDetailsRequest,
            referenceTransactionDetails)
    } catch (fiservTTPCardReaderError:FiservTTPCardReaderException) {
        // TODO handle error
    }
}
```

No reference transaction identifier is required to perform an open refund.

```java
val amount = 10.99
val merchantOrderId = "merchantOrderId"
val merchantTransactionId = "merchantTransactionId"
val captureFlag = "true"
val createToken = "true"

val transactionDetailsRequest: TransactionDetailsRequest = TransactionDetailsRequest(
    captureFlag,
    merchantTransactionId,
    merchantOrderId,
    createToken )

val referenceTransactionDetails: ReferenceTransactionDetails = ReferenceTransactionDetails(
    referenceMerchantTransactionId )

CoroutineScope(Dispatchers.Main).launch {
    try{
        FiservTTPCardReader.refunds(
            amount.toBigDecimal(),
            RefundTransactionType.OPEN,
            transactionDetailsRequest,
            referenceTransactionDetails)
    } catch (fiservTTPCardReaderError:FiservTTPCardReaderException) {
        // TODO handle error
    }
}
```

## See also

- [Commerce Hub Tap to Pay on Android Integration Guide](https://developer.fiserv.com/product/CommerceHub/docs/?path=docs/In-Person/Integrations/Mobile-SDK/Android-TTP.md&branch=preview)

- [Commerce Hub Sample App](https://github.com/Fiserv/ch-ttp-android-sample-app)

---
