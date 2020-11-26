# ps-performance-api
A API for performance test based on the [performance tool project](https://github.com/wexinc/ps-performance-tool).

## How does it work?

The API is capable of different types of test involving different message protocols including Wex's proprietary formats. Those types of tests are defined by the path.

Each type of test has its own properties but some are common to all of them. All tests require an unique id that can be used to retrieve the test results and a sequence of steps that act as instructions. For very simple scenarios only one step is necessary, for more complex ones we need more steps. 

### Simple test

This is the most basic type of test, where we define just the number of threads and requests sent for each thread.  

The simple test can be run by sending a POST request to host:7000/run/simple with a configuration JSON as body. An example of a body would be:

```json
{  
    "id" : "MyFirstTest",
    "thread_count":5,
    "request_count":10,
    "thread_delay":100,
    "steps" : [
		{
			"name" : "MyRequest",
			"type" : "sender_http",
            "uri":"http://testsite.com:8080/example/",
			"persistent":false,
			"method" : "GET",
			"template" : "{\"transaction_id\":\"${generate_alpha_numeric(22)}\",\"merchant_prefix\":\"XI\",\"amount\":\"2.55\",\"myID\":\"192940632884\"}",
			"headers" : {
				"Content-Type" : "application/json"
			},
			"expected_status_code": 200            
		}
   ]
}
```

### Load test

This is a type of test based on the desire throughput, the test can maintain the throughput fixed for the whole test time or it can be incremental reaching the peak within a defined time interval. The load test can be run by sending a POST request to host:7000/run/load. An example of a json body would be:

```json
{  
    "id" : "MY_LOAD",
    "request_per_second":20,
    "test_time_seconds": 30,    
    "max_thread":120,
    "steps" : [
		{
			"name" : "authorization request",
			"type" : "ISO8583_sender_step",
			"template":"<isomsg>
			  <!-- org.jpos.iso.packager.XMLPackager -->
			  <field id=\"0\" value=\"0100\"\/>
			  <field id=\"2\" value=\"690046040123456789\"\/>
			  <field id=\"3\" value=\"000000\"\/>
			  <field id=\"4\" value=\"000000000100\"\/>
			  <field id=\"7\" value=\"1029135914\"\/>
			  <field id=\"9\" value=\"12345678\"\/>
			  <field id=\"10\" value=\"50112345\"\/>
			  <field id=\"11\" value=\"${generate_numeric(6)}\"\/>
			  <field id=\"12\" value=\"135914\"\/>
			  <field id=\"13\" value=\"1029\"\/>
			  <field id=\"14\" value=\"2003\"\/>
			  <field id=\"15\" value=\"1020\"\/>
			  <field id=\"16\" value=\"1029\"\/>
			  <field id=\"18\" value=\"5542\"\/>
			  <field id=\"22\" value=\"000\"\/>
			  <field id=\"32\" value=\"123654\"\/>
			  <field id=\"33\" value=\"123654\"\/>
			  <field id=\"37\" value=\"${generate_numeric(12)}\"\/>
			  <field id=\"43\" value=\"          GENERIC NAME  GENERIC CITY USA\"\/>
			  <field id=\"48\" value=\"P6315${generate_numeric(15)}421901032110203222030109203012\"\/>
			  <field id=\"120\" value=\"012988310    XYZ STREET          \"\/>
			<\/isomsg>",
			"host":"test.us-east-1.dev.com",
			"port":10003,
			"persistent":true,
			"expected_result":"00",
			"packager":"MASTERCARD",
			"unique_field":"11",
			"has_length_header":"false"
		}
   ]
}
```

### Step

Every step has a name and the type. Depending on the type further configuration is necessary. The steps can be a Sender, Timer or Processor.

| Category  | Type                       | Description
| ---       | ---                        | --- 
| Sender    | ISO8583_sender_step        | Sends an ISO8583 message
| Sender    | TCP_raw_sender_step        | Sends a Raw Byte array message over TCP
| Sender    | HTTP_sender_step           | Sends a HTTP message
| Timer     | Gaussian_timer_step        | Make each thread waits depending on a gaussian function
<!---| Timer     | Relative_timer_step                |-->
| Processor | ISO8583_response_extractor_step | Extract a value from a ISO8583 field in the response
| Processor | Regex_response_extractor_step   | Extract a value from a string response using regex
| Processor | Parameter_writer_step           | Store a given value into a parameter to be used in other step 

#### Sender 

This kind of step is able to send messages from different protocols depending on its type. Currently, there are three type of senders: 

* ISO8583 Sender (ISO8583_sender_step)

| Parameter       | Type    | Mandatory | Description
| ---             | ---     | ---       | ---  
|name             | `string`| false     | identifier of the step
|type             | `string`| true      | fixed value "ISO8583_sender_step"
|template         | `string`| true      | the content of the message in XML format
|host             | `string`| true      | the endpoint where the message will be sent
|port             | `int`   | true      | the destination port 
|persistent       | `string`| true      | flag to indentify if the connection will be closed after each message
|expected_result  | `string`| true      | the expected value for field 39 on response
|packager         | `string`| true      | the ISO packager used to format the message
|unique_field     | `string`| true      | an ISO field that can be used as an unique identifier (this have to be echoed in the response)
|has_length_header| `string`| true      | identify if the message requires a length header

* TCP Sender (TCP_raw_sender_step)

| Parameter       | Type    | Mandatory | Description
| ---             | ---     | ---       | ---  
|name             | `string`| false     | identifier of the step
|type             | `string`| true      | fixed value "TCP_raw_sender_step"
|template         | `string`| true      | the content of the message in ASCII
|host             | `string`| true      | the endpoint where the message will be sent
|port             | `int`   | true      | the destination port 
|persistent       | `string`| true      | flag to indentify if the connection will be closed after each message
|expected_result  | `string`| true      | a regex that needs to match a substring of the response 
|stop_character   | `string`| true      | the character which identifies the end of message

* HTTP Sender (HTTP_sender_step)

| Parameter       | Type              | Mandatory | Description
| ---             | ---               | ---       | ---  
|name             | `string`          | false     | identifier of the step
|type             | `string`          | true      | fixed value "HTTP_sender_step"
|template         | `string`          | true      | the raw body of the message
|uri              | `string`          | true      | the URI where the message will be sent
|method           | `string`          | true      | the HTTP method
|headers          | `<*>`             | false     | the headers for the HTTP message
|query_parameters | `<*>`             | false     | the query parameters for the HTTP message
|expected_body_pattern  | `string`| true | a regex that needs to match a substring of the response body
|expected_status_code   | `string`| true | the expected status code 

#### Timer

The timer is a step in which the thread will be suspended for a scpecified amount of time.

* Gaussian Timer (Gaussian_timer_step)

| Parameter       | Type           | Mandatory | Description
| ---             | ---            | ---       | ---  
|name             | `string`       | false     | identifier of the step
|type             | `string`       | true      | fixed value "Gaussian_timer_step"
|min_value        | `int`          | true      | The min value for the gaussian function
|max_value        | `int`          | true      | The max value for the gaussian function

#### Processor 

* ISO8583 response extractor (ISO8583_response_extractor_step)

| Parameter       | Type           | Mandatory | Description
| ---             | ---            | ---       | ---  
|name             | `string`       | true      | the name of the parameter that will store the extracted value
|type             | `string`       | true      | fixed value "ISO8583_response_extractor_step"
|field            | `string`       | true      | the data element to be extracted

* Regex response extractor (Regex_response_extractor_step)

| Parameter       | Type           | Mandatory | Description
| ---             | ---            | ---       | ---  
|name             | `string`       | true      | the name of the parameter that will store the extracted value
|type             | `string`       | true      | fixed value "Regex_response_extractor_step"
|regex            | `string`       | true      | a regex with a capture group to extract the value

* Parameter writer (Parameter_writer_step)

| Parameter       | Type           | Mandatory | Description
| ---             | ---            | ---       | ---  
|name             | `string`       | true      | the name of the parameter that will store the value
|type             | `string`       | true      | fixed value "Parameter_writer_step"
|value            | `string`       | true      | the value to be stored
|has_regex        | `boolean`      | true      | identify if the value has a function 



----------------------
${generate_numeric(12)}

"packager":"MASTERCARD|CT7",

TODO: 

 - CT7 packager should be renamed


