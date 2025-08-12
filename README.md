## Implementing a Custom UI Card Payment System with Payabl Authorization and 3D Secure

This guide provides a comprehensive walkthrough for creating a secure card payment system with a custom user interface using payabl's authorization method and 3D Secure verification in a sandbox environment.

### 1. System Overview and Integration Method

For a fully custom user interface where you collect card details directly, you will use **payabl's Direct Integration (Server-to-Server API)**. This method provides complete control over the checkout experience. However, it's crucial to understand that this requires a high level of PCI DSS compliance as your server will handle sensitive cardholder data.

The payment flow will be as follows:
1.  The customer enters their card details on your custom UI.
2.  Your frontend securely sends this data to your backend server.
3.  Your backend server sends an authorization request to the payabl. API.
4.  If 3D Secure is required, payabl. returns a redirection URL.
5.  Your application redirects the customer to their bank's 3D Secure page.
6.  After the customer completes the 3D Secure challenge, they are redirected back to your website.
7.  payabl. sends a final transaction status notification to your backend.

### 2. Sandbox Environment and Credentials

All testing will be conducted in the payabl. sandbox environment.

*   **Sandbox API Endpoint:** `https://sandbox.payabl.com/pay/backoffice/payment_authorize`
*   **Sandbox Backoffice:** `https://portal.sandbox.payabl.com/`
*   **Test Merchant ID:** `gateway_test_3d`
*   **Test Secret Key:** `b185`

### 3. Required Parameters for the Authorization Request

Your custom UI will need to collect the following card and customer information to be sent in the POST request to the payabl. API:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `merchantid` | String | Yes | Your merchant identification number. Use `gateway_test_3d` for testing. |
| `amount` | Float | Yes | The transaction amount. |
| `currency` | String | Yes | The 3-letter ISO 4217 currency code (e.g., EUR, USD). |
| `orderid` | String | No | A unique identifier for the order from your system. |
| `payment_method` | Integer | Yes | Set to `1` for Credit Card. |
| `ccn` | String | Yes | The customer's credit card number. |
| `exp_month` | String | Yes | The 2-digit expiry month of the card (e.g., '01'). |
| `exp_year` | String | Yes | The 4-digit expiry year of the card (e.g., '2025'). |
| `cvc_code` | String | Yes | The 3 or 4-digit card verification code. |
| `cardholder_name` | String | Yes | The name of the cardholder as it appears on the card. |
| `email` | String | Yes | The customer's email address. |
| `customerip` | String | Yes | The customer's IP address. |
| `param_3d` | String | Yes | Specifies the 3D Secure requirement. Use `always3d` to enforce it. |
| `url_return` | String | Yes | The URL on your site where the customer is redirected after the 3D Secure process. |
| `notification_url` | String | Yes | The URL on your server where payabl. will send the final transaction status notification. |
| `signature` | String | Yes | The calculated SHA-1 signature for the request. |

### 4. Signature Calculation

Every request to the payabl. API must be signed to ensure its integrity.

**Steps to calculate the signature:**
1.  Create a collection of all the request parameters and their values.
2.  Sort these parameters alphabetically by their names.
3.  Concatenate the *values* of the sorted parameters into a single string.
4.  Append your secret key (`b185` for testing) to the end of this string.
5.  Calculate the SHA-1 hash of the resulting string.
6.  The final signature must be a lowercase hexadecimal string.

**Important:** The parameter values must **not** be URL-encoded before the signature calculation. If the signature is incorrect, you will likely receive an error with code -999 or -6000.

### 5. Implementation Guide: Frontend and Backend

#### **Frontend Implementation (Your Custom UI)**

1.  **Build the Payment Form:** Create an HTML form with input fields for all the necessary card details (`ccn`, `exp_month`, `exp_year`, `cvc_code`, `cardholder_name`) and any other required customer information.
2.  **Securely Submit to Your Backend:** On form submission, send the collected data to your backend server over HTTPS. **Never send card details directly from the client-side to the payabl. API.**
3.  **Handle the 3D Secure Redirect:** Your backend will provide a `url_3ds`. Your frontend must redirect the user to this URL.
    ```javascript
    // Example in JavaScript after receiving the url_3ds from your backend
    window.location.href = url_3ds;
    ```
4.  **Create a Return Page:** This is the page at the `url_return` you specified. It should inform the user that their payment is being processed and that they will be notified of the final status. The definitive result comes via the backend notification.

#### **Backend Implementation**

1.  **Create an Authorization Endpoint:** This endpoint on your server will receive the payment details from your frontend.
2.  **Construct the Authorization Request:**
    *   Assemble all the required parameters.
    *   Calculate the `signature` as described above.
3.  **Send the Request to Payabl:** Make a POST request to `https://sandbox.payabl.com/pay/backoffice/payment_authorize` with the request body containing all parameters.
4.  **Handle the Initial Response:**
    *   **If 3D Secure is Required:** The response will have a `status` of `2000` (pending) and will include a `url_3ds` parameter. Your server should send this `url_3ds` back to your frontend to trigger the redirect.
    *   **If there is an Error:** The response will contain an error code and message. Log this and inform the user.
5.  **Create a Notification Endpoint:** This is the `notification_url` you provided. It will receive a POST request from payabl. with the final transaction outcome.
6.  **Verify the Notification:** The notification will contain parameters such as `transactionid`, `type`, `errorcode`, and `timestamp`. You must verify the authenticity of this notification by calculating a simplified signature using these parameters and your secret key.
7.  **Process the Final Status:**
    *   Check the `errorcode` in the notification. An `errorcode` of `0` generally indicates a successful transaction.
    *   Update your database with the final status of the order.
    *   Notify the customer of the final outcome (e.g., via email).

### 6. Verifying Transaction Statuses in Sandbox

Use the following test cards to simulate different 3D Secure scenarios:

| Scenario | Card Number | Expected Behavior | Final Status |
|---|---|---|---|
| **Success** | `4242424242424242` | The user is redirected to a 3D Secure page and the authentication is successful. | **Success** |
| **Failure** | `4242000000000000` | The user is redirected to a 3D Secure page, but the authentication fails. | **Failure** |
| **Pending then Success** | - | Any transaction requiring 3D Secure will initially have a pending status (`status=2000`). After the user successfully completes the 3D Secure challenge, a success notification is sent. | **Pending -> Success** |
| **Pending then Failure** | - | A transaction will be pending while the user is completing the 3D Secure challenge. If they fail, a failure notification is sent. | **Pending -> Failure** |
| **Card Not Enrolled** | `4012001038443335` | The card is not enrolled in 3D Secure. The transaction may be declined depending on your `param_3d` setting. | **Failure** |
| **Unable to Verify** | `4012001038488884` | The enrollment status cannot be verified. | **Failure** |
| **Soft Decline** | `5406592832271063` | For a `non3d` transaction attempt, the issuer responds with a "soft decline" (error code 65), and you will receive a `url_3ds` to proceed with 3D Secure. | **Failure -> Pending -> Success/Failure** |
