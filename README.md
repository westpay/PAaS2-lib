### Foundation
**Launch of PAaS application**

Due to external requirements (PCI, Card brands etc) we hxave had to lock down the Android system. 
Due to this the terminal user (merchant) will not be able to manually start any applications.

The Payment application is responsible for starting the PAaS application.
For the Payment application to be able to start the PAaS application, the PAaS application package name and class name of the Activity to be started.

The PAaS application needs to define below action in the intent-filter for the Activity to be started, this way the Payment application will be able to find the PAaS application package to be started.  
```java 
<action android:name="se.westpay.paaslib.SIGNATURE" /> 
``` 
Example of an Activity with intent-filter.

```xml
 <activity
            android:name=".MainActivity"
            android:exported="true"
            android:label="@string/app_name"
            android:theme="@style/Theme.OnlineApiClient.NoActionBar">
            <intent-filter>
                <action android:name="se.westpay.paaslib.SIGNATURE" />
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```

### Instantiate PAaS library

It is very likely that the PAaS library needs to be accessible in different contexts. 
One solution to this is to extend the Android Application class [Application](https://developer.android.com/reference/android/app/Application) 
and create an instance of the CarbonTerminal PAaS library class in the extended Application class.

Below is an example of this.

```kotlin
class MyApplication : Application() {

    companion object {
        lateinit var paasAppLib : CarbonTerminal
    }

    override fun onCreate() {
        super.onCreate()
        paasAppLib = CarbonTerminal()
    }
}
```

### Exit your PAaS application

The 'notifyExit' function should be executed before exit the client application to notify the exit action to the PA.

PAaS library will start PA main activity once this function is called.

```kotlin
fun notifyExit(context: Context): Boolean
```

Below is an example of executing the notifyExit function.

```kotlin
// Notifying the application exit action to the payment application.
val paasApp = (requireActivity().applicationContext) as PAaSApplication
context?.let { paasApp.getData()?.notifyExit(it) }
    ?: _log.error("Unable to notify the application exit.")

(requireActivity() as MainActivity).finishAffinity()
			
```
___

## Examples

### Basic payment

**Step 1**

Before one can start to use the PAaS library functions, the PAaS library payment service message handler needs to be instantiated, this is done by using the PAaS library function register

```kotlin
/**
 * Registers the payment terminal, which involves creating resources needed by the terminal API.
 *
 * @param context android.content.Context
 * @param response Handler that get called when a connection to the payment service has been completed
 * @return Boolean - True successful operation, false otherwise.
 */
fun register(context: Context?, response: RegisterResponseHandler? = null) : Boolean
```
___

**Step 2**

After a successful register call, the Payment Application service need to be setup and ready to be able to handle transaction requests from the PAaS application. 
This is done by performing the initialise call.

```kotlin
/**
 * Initialises the payment terminal. Before the payment terminal can be used to process
 * transactions, it has to be initialised. The initialisation process includes contacting the
 * payment processor and downloading the current terminal settings.
 *
 *
 * This method returns immediately but the initialisation result is returned in the response
 * callback. The initialisation takes a little time, from a few seconds to a couple of minutes,
 * depending on network availability and if there are new settings or updates to process.
 *
 *
 * The initialisation is done in the background. The payment application will not take over
 * the screen during this time.
 *
 * @param settings     The settings for the payment terminal.
 * @param response     A callback that will be called with the result of the initialisation.
 * @param notification Optional. A callback that will be called for different events.
 * @return `true` if the request was sent successfully, otherwise `false`.
 * Note that this does not indicate the success or failure of the initialisation.
 * If no `InitialisationResponseHandler` is provided the method will return false.
 */
fun initialise(
    settings: TerminalSettings,
    response: InitialisationResponseHandler,
    notification: NotificationHandler?
): Boolean
```

The *initialise* function will not result in the Payment Application service taking over the display. 
So bringing the PAaS application to front of the Activity back stack is not needed when performing the *initialise* call.

When performing the initialise call, the Payment Application service will perform a number of network related steps in the background that might take some time.

1. The first step the PA will do is to download and install configuration information if required.
2. The second step is to perform a logon to the authorization host.

The PA will not be ready to accept payment requests before above two steps is successfully completed.

**Regarding the arguments in the *initialise* function.**

The first argument is TerminalSettings, here the terminal id and the configuration- and authorization hosts are provided to the Payment Application service.

example values:

| parameter          | value               | comment                          |
|--------------------|---------------------|----------------------------------|
| Terminal-id        | 80000001            | The ID is always 8 digits long   |
| Configuration host | 185.27.171.42:55133 | This is for the test environment |
| Authorization host | 185.27.171.42:55144 | This is for the test environment |

The second argument is a InitialisationResponseHandler. 
The PAaS application needs to define and implement a InitialisationResponseHandler that handles the response from the initialise call.

Implementation of a NotificationHandler is optional. 
The NotificationHandler provide notifications during logon to authorization server, performing transaction, etc. 
Important when receiving notifications is that the PAaS application does not bring it self to the front of the Activity back stack, 
since this will disturb the work of the Payment Application service.

___

**Step 3**

After a successful initialise call, one can start to perform transactions. 

First a [ActivityResultLauncher](https://developer.android.com/reference/androidx/activity/result/ActivityResultLauncher) needs to be defined.
 ```kotlin
 private lateinit var _transactionResultLauncher: ActivityResultLauncher<Intent>
 ```


To be able to handle the result from a transaction a [ActivityResultContracts](https://developer.android.com/reference/androidx/activity/result/contract/ActivityResultContracts) 
needs to be registered using the 
[registerForActivityResult](https://developer.android.com/reference/androidx/activity/result/ActivityResultCaller#registerForActivityResult(androidx.activity.result.contract.ActivityResultContract%3CI,O%3E,androidx.activity.result.ActivityResultCallback%3CO%3E))
call. The [TransactionResponse] can be obtained by using the [PaymentTerminal.createTransactionResponseFromIntent] call.
  ```kotlin
  _transactionResultLauncher = 
  registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            val paasApp = ((requireActivity() as MainActivity).application) as PaasApplication
            val response: TransactionResponse = paasApp.getData().createTransactionResponseFromIntent(result.data)
        }
  }
 ```


When it's time to perform the transaction a [Intent](https://developer.android.com/reference/android/content/Intent) 
for the transaction needs to be created.
This can be done by using the method [PaymentTerminal.createIntentForTransaction] and provide a PurchaseRequest object 
as argument.

 ```kotlin
  val purchaseRequest = PurchaseRequest(currencyId = Constants.CURRENCY_ID_NUMBER_DEFAULT,
                                        totalAmount = 10000,
                                        saleId = "1",
                                        loyaltyApplications =  emptyArray(),
                                        paymentMethod = "",
                                        cashBackAmount = 0,
                                        tipAmount = 0,
                                        maximumTipAmount = 0,
                                        maximumTipPercent = 100,
                                        allowSkipTip = SkipTipOptions.NO)
  purchaseRequest.setDisabledFeatures(arrayOf<PaymentFeatures>())

  val paasApp = ((requireActivity() as MainActivity).application) as PaasApplication
  val intent: Intent = app.getData().createIntentForTransaction(purchaseRequest)
 ```


Then by using the [Intent](https://developer.android.com/reference/android/content/Intent) 
it's possible to launch the payment application and wait for the result callback to be called.
 ```kotlin
 _transactionResultLauncher.launch(intent)
 ```

___

**Step 4**

After a completed transaction a receipt should be printed.

A receipt should include merchant information, date of the payment and details of the payment. 
The information concerning the details of the payment depends on the type of payment method used for the payment.

If a payment card has been used for the payment, below information should at least be present on the receipt.

Reference number
Verification and authorisation method used
Card name
If contactless or chip card was used
Denial reason if declined payment
EMV data, as AID, TVR and TSI
If an alternative payment has been used, below information should at least be available on the receipt.

Reference number
Payment method
To print a receipt in a PAaS application start by calling the function startPrintJob.

```kotlin
/**
 * Starts a new printer job.
 *
 * @return A printer job instance, or null if a printer job cannot be started at this moment.
 */
fun startPrintJob(context: Context): PrintJob
```

The function returns a PrintJob that the PAaS application use to print a receipt and wait for the printing to finish. 
The format of the receipt supported by the PrintJob are text and bitmap image.

Below is the function to print a text receipt.
```kotlin
/**
 * Prints one line of text. Text that does not fit on the line is truncated.
 *
 * @param left Text that should be left-aligned.
 * @param center Text that should be center-aligned.
 * @param right Text that should be right-aligned.
 * @param monospace True if the text should be printed with a monospace font. False
 * if the text should be printed with a variable-width font.
 * @param bold True if the text should be printed in bold-face.
 */
fun printTextLine(
    left: String?,
    center: String?,
    right: String?,
    monospace: Boolean,
    bold: Boolean
): Boolean
```

Below is the function to print a bitmap image receipt.
```kotlin
/**
 * Prints a bitmap.
 *
 * @param image A bitmap to print.
 * @return True if successful, otherwise false.
 */
fun printImage(image: Bitmap?): Boolean
```

Below is an example of print a bitmap image receipt.
```kotlin
class PrinterTestViewModel : ViewModel() {

    private val _printResult = MutableLiveData<Boolean>()
    val printResult: LiveData<Boolean> = _printResult

    /**
     * Print image.
     *
     * @param context
     */
    fun printImage(context: Activity) {
        viewModelScope.launch {
            val success = withContext(Dispatchers.Default) {
                var success = false
                val image = BitmapFactory.decodeResource(context.resources, R.drawable.small)
                val paasApp = (context.application) as PaasApplication
                val printJob: PrintJob = paasApp.getData().startPrintJob(context)
                try {
                    success = printJob.printImage(image)
                    if (success) {
                        val printerStatus: PrinterStatus = printJob.finish()
                        Log.i(Constants.TAG, "Printer finished, status: " + printerStatus.name)
                        if (printerStatus == PrinterStatus.OK) {
                            success = true
                        }
                    } else {
                        Log.i(Constants.TAG, "Failed to print image")
                    }
                } catch (e: Exception) {
                    Log.e(Constants.TAG, e.message!!)
                }
                success
            }
            _printResult.postValue(success)
        }
    }
}
```

