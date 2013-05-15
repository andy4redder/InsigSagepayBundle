A Symfony 2 bundle to manage SagePay integration
================================================

Quick How-To
------------

## Setup ##

### Add config to config yaml ###

```
insig_sagepay:
    vendor: vendorname
    mode: simulator
    redirect_urls:
        ok: routename
        notauthed: routename
        abort: routename
        rejected: routename
        authenticated: routename
        registered: routename
        error: routename
        invalid: routename
        fail: routename
        malformed: routename
        token_ok: routename
        token_error: routename
```

### Add to AppKernel ###

```
...
new Insig\SagepayBundle\InsigSagepayBundle(),
...
```

## Use ##

### Grab the service ###

```
	$spm = $this->get("insig_sagepay.manager")
```

### Build Transaction Registration Request ###

```
	$pr = new TransactionRegistrationRequest();
	$pr->setAmount(100)
	$pr->setCurrency("GBP")
	$pr->setDescription("Description")
	$pr->setNotificationUrl("http://url.com)
	etc...

```

### Grab your SagePay Payment Transaction Entity ###

```
	$sagepayPayment = new SagePayPayment();
```

### Register the transaction ###

```
	$transactionRegistrationResponse = $spm->registerTransaction($pr, $sagepayPayment);
```

### Build your confirmation form ###

Set the form action to the next url from the Transaction Registration Response

```
	<form action="{{ transactionRegistrationResponse.nextUrl }}" method="POST">
			<input type="submit" value="Confirm and Pay" class="submit" />
	</form>
```

### Build Notification URL

StatusDetail is required however not always posted.
SagePay manager requires raw POST data.

```
	$spm = $this->get("insig_sagepay.manager");
    $postData = file_get_contents("php://input");
    if (!strstr($postData, "StatusDetail")) { $postData .= "&StatusDetail=OK"; }
```

### Create Transaction Notification

```
	$transactionNotification = $sagepayManager->createTransactionNotification($postData);
```

### Fetch payment details from database

Check response is authentic using your security key

```
	$sagepayPayment = $this->getDoctrine()->getEntityManager()->getRepository("SagePayPayment")->findOneByVendorVps($transactionNotification->getVendorTxCode(), $transactionNotification->getVpsTxId());
    $isAuthentic = $sagepayManager->isNotificationAuthentic($transactionNotification, $sagepayPayment->getSecurityKey());
```


### Build response ###

If response is authentic, save your response to your SagePay Payment Entity and return an OK.

```
    $this->updatePaymentFromNotification($sagepayPayment, $transactionNotification);
    $transactionNotificationResponse = $sagepayManager->createTransactionNotificationResponse("OK", NULL);
    $formattedResponse = $sagepayManager->formatNotificationResponse($transactionNotificationResponse);

```

Otherwise, return INVALID

```
	$transactionNotificationResponse = $sagepayManager->createTransactionNotificationResponse("INVALID", "Signature does not match");
    $formattedResponse = $sagepayManager->formatNotificationResponse($transactionNotificationResponse);
```

```
	$response = new Response();
    $response->setContent($formattedResponse);
    $response->setStatusCode(200);
    $response->headers->set("Content-Type", "text/plain");
    return $response;
```



