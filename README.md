# Step by Step Guide to Integrating SSLCommerz Payment Gateway in PHP/Laravel Framework

![1707763000403](https://github.com/kaamrul/laravel-sslcommerz/assets/37633219/918daf7d-3375-4f53-8988-1ec738433667)


## Installation


### Step 1: Download SSLCommerz - Laravel Library from github. After download extract the library files. [Click Here](https://github.com/sslcommerz/SSLCommerz-Laravel)

### Step 2: Copy the Library folder and put it in the laravel project's app/ directory. If needed, then run composer dump -o.

### Step 3: Copy the config/sslcommerz.php file into your project's config/ folder.

### Step 4: Update config/session.php file into your project's config/ folder.

```bash
'secure' => env('SESSION_SECURE_COOKIE'), to 'secure' => env('SESSION_SECURE_COOKIE', true), 

'same_site' => 'lax', to 'same_site' => 'none', or 'same_site' => 'null',
```


### Step 5: Configure SSLCommerz Credentials inside .env file add 3 key-value pairs

```
SSLCZ_STORE_ID=<your store id>
SSLCZ_STORE_PASSWORD=<your store password>
SSLCZ_TESTMODE=true // set false when using live store id
```
For development purposes, you can obtain sandbox 'Store ID' and 'Store Password'
by registering at https://developer.sslcommerz.com/registration/


###  Step 6: Add exceptions for VerifyCsrfToken middleware accordingly (you actual routes) Like this - 

```bash
protected $except = [
    '/success',
    '/cancel',
    '/fail',
    '/ipn',
    '/place-order',
];
```

###  Step 7: Now, let's create a  PaymentController inside your app/Http/Controllers  controller to handle payment requests. Run the following command to generate a new controller - 

```bash
php artisan make:controller PaymentController
```
Open the PaymentController.php file located in app/Http/Controllers directory and add the following method - 

```bash
<?php
namespace App\Http\Controllers;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use App\Library\SslCommerz\SslCommerzNotification;
class PaymentController extends Controller
{
    public function placeOrder(Request $request)
    {
        # Here you have to receive all the order data to initate the payment.
        # Lets your oder trnsaction informations are saving in a table called "orders"
        # In orders table order uniq identity is "transaction_id","status" field contain status of the transaction, "amount" is the order amount to be paid and "currency" is for storing Site Currency which will be checked with paid currency.
        $cart_data = json_decode($request->cart_json, true);
        $cart_data['transaction_id'] = uniqid();
        $total_amount = $cart_data['orderTotalPrice'];
        if ($total_amount < 10) {
            return response()->json(['success' => false, 'message' => 'Opps! something went wrong']);
        }
        $post_data = array();
        $post_data['total_amount'] = $total_amount; # You cant not pay less than 10
        $post_data['currency'] = "BDT";
        $post_data['tran_id'] = $cart_array['transaction_id']
        # CUSTOMER INFORMATION
        $post_data['cus_name'] = 'Customer Name';
        $post_data['cus_email'] = 'customer@mail.com';
        $post_data['cus_add1'] = 'Customer Address';
        $post_data['cus_add2'] = "";
        $post_data['cus_city'] = "";
        $post_data['cus_state'] = "";
        $post_data['cus_postcode'] = "";
        $post_data['cus_country'] = "Bangladesh";
        $post_data['cus_phone'] = '8801XXXXXXXXX';
        $post_data['cus_fax'] = "";
        # SHIPMENT INFORMATION
        $post_data['ship_name'] = "Store Test";
        $post_data['ship_add1'] = "Dhaka";
        $post_data['ship_add2'] = "Dhaka";
        $post_data['ship_city'] = "Dhaka";
        $post_data['ship_state'] = "Dhaka";
        $post_data['ship_postcode'] = "1000";
        $post_data['ship_phone'] = "";
        $post_data['ship_country'] = "Bangladesh";
        $post_data['shipping_method'] = "NO";
        $post_data['product_name'] = "Computer";
        $post_data['product_category'] = "Goods";
        $post_data['product_profile'] = "physical-goods";
        #Before  going to initiate the payment order status need to update as Pending.
        // Store data in Order table and others table
        $orderData = [
            'transaction_id'       => $cart_data['transaction_id'],
            'customer_id'          => $cart_data['customer_id'],
            'address_id'           => $cart_data['shippingAddress'],
            'quantity'             => $cart_data['total_quantity'],
            'sub_total_amount'     => $cart_data['subtotalPrice'],
            'total_amount'         => $cart_data['orderTotalPrice'],
            'shipping_cost'        => $cart_data['shippingPrice'],
            'payment_status'       => 'Pending',
        ];
        Order::create($orderData);
        // Store data into others table if you have
        $sslc = new SslCommerzNotification();
        # initiate(Transaction Data , false: Redirect to SSLCOMMERZ gateway/ true: Show all the Payement gateway here )
        $payment_options = $sslc->makePayment($post_data, 'checkout', 'json');
        if (!is_array($payment_options)) {
            print_r($payment_options);
            $payment_options = array();
        }
    }
}
```

###  Step 8: Handle Payment Callbacks in the PaymentController, define methods to handle payment success, fail, cancel and ipn callbacks - 

```bash
public function success(Request $request)
{
      $tran_id = $request->input('tran_id');
      $amount = $request->input('amount');
      $currency = $request->input('currency');
      $sslc = new SslCommerzNotification();
      #Check order status in order tabel against the transaction id or order id.
      $order_details = DB::table('orders')
          ->where('transaction_id', $tran_id)
          ->first();
      if ($order_details->payment_status == 'Pending') {
          $validation = $sslc->orderValidate($request->all(), $tran_id, $amount, $currency);
          if ($validation) {
              $update_product = DB::table('orders')
                  ->where('transaction_id', $tran_id)
                  ->update(['payment_status' => 'Complete']);
              return redirect()->route('invoice')->with('success', 'Order is Successfully Placed');
          }
      } else if ($order_details->status == 'Processing' || $order_details->status == 'Complete') {
          /*
            That means through IPN Order status already updated. Now you can just show the customer that transaction is completed. No need to udate database.
          */
          return redirect()->route('invoice')->with('success', 'Order is Successfully Placed');
      } else {
          #That means something wrong happened. You can redirect customer to your product page.
          return redirect()->route('invoice')->with('error', 'Invalid Transaction');
      }
}
public function fail(Request $request)
{
  // Handle failed payment
}
public function cancel(Request $request)
{
  // Handle cancelled payment
}
public function ipn(Request $request)
{
  // Handle ipn payment
}
```

###  Step 9: Create following route inside your routes/web.php  Ex - 

```bash
Route::post('/place-order', [PaymentController::class, 'placeOrder']);
Route::post('/success', [PaymentController::class, 'success']);
Route::post('/fail', [PaymentController::class, 'fail']);
Route::post('/cancel', [PaymentController::class, 'cancel']);
Route::post('/ipn', [PaymentController::class, 'ipn']);
```
import PaymentController into your web.php file also

```bash
use App\Http\Controllers\PaymentController;
```

###  Step 10: To Show sslcommerz gateway page inside a popup

To integrate it you need to have a <button> with following properties in your blade file -
```bash
<button id="sslczPayBtn"
    token="if you have any token validation"
    postdata=""
    order="If you already have the transaction generated for current order"
    endpoint="/place-order"> Place Order
</button>
```

Populate postdata obj as required -
```bash
<script>
    var obj = {};
    obj.cus_name = $('#customer_name').val();
    obj.cus_phone = $('#mobile').val();
    obj.cus_email = $('#email').val();
    obj.cus_addr1 = $('#address').val();
    obj.amount = $('#total_amount').val();
    $('#sslczPayBtn').prop('postdata', obj);
</script
```

Add following script -
#### Common Script
```bash
<link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css"
integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">

<script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js"
integrity="sha384-JjSmVgyd0p3pXB1rRibZUAYoIIy6OrQ6VrjIEaFf/nJGzIxFDsf4x0xIM+B07jRM"
crossorigin="anonymous"></script>
```
#### For Sandbox
```bash
<script>
    (function (window, document) {
        var loader = function () {
            var script = document.createElement("script"), tag = document.getElementsByTagName("script")[0];
            script.src = "https://sandbox.sslcommerz.com/embed.min.js?" + Math.random().toString(36).substring(7);
            tag.parentNode.insertBefore(script, tag);
        };
        window.addEventListener ? window.addEventListener("load", loader, false) : window.attachEvent("onload", loader);
    })(window, document);
</script>
```
#### For Sandbox
```bash
<script>
    (function (window, document) {
        var loader = function () {
            var script = document.createElement("script"), tag = document.getElementsByTagName("script")[0];
            script.src = "https://sandbox.sslcommerz.com/embed.min.js?" + Math.random().toString(36).substring(7);
            tag.parentNode.insertBefore(script, tag);
        };
        window.addEventListener ? window.addEventListener("load", loader, false) : window.attachEvent("onload", loader);
    })(window, document);
</script>
```

### Step 11: Test Your Integration

By following these steps, you can securely process online payments and enhance the functionality of your Laravel-based e-commerce platform or web application. 


## License

This repository is licensed under the [MIT License](http://opensource.org/licenses/MIT).

Copyright 2024 [Md Kamrul Hasan](https://github.com/kaamrul). We are not affiliated with SSLCommerz and don't give any guarantee. 
