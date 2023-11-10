# **Tehokas cURL API-integraatio PHP:ssa: RESTful-toiminnot JSON-datalla käsittelyssä**

Viimeisimmässä projektissani kohtasin tehtävän integroida ulkoinen API saumattomasti käyttäen cURL-pyyntöjä PHP:ssa. Liikkuessani tämän prosessin läpi ensimmäistä kertaa, kohtasin haasteita ymmärtää sen monimutkaisuuksia. Helpottaakseni tulevia ponnistuksiani ja mahdollisesti auttaakseni muita, dokumentoin cURL API-kutsuni tässä postauksessa.

Tässä annetut esimerkit ja funktiot ovat yhteensopivia PHP-version 5.6+ kanssa.

## PHP cURL-perusteiden ymmärtäminen

cURL, lyhenne sanoista 'Client URL Library', mahdollistaa yhteyden muodostamisen ja kommunikoinnin erilaisten palvelinten kanssa käyttäen erilaisia protokollia (HTTP, HTTPS, FTP, proxy, evästeet jne.). Syventyäksesi cURL:n toimintaan, tutustu [viralliseen PHP-dokumentaatioon](https://www.php.net/manual/en/intro.curl.php). Tämä artikkeli pyrkii tarjoamaan kattavampia esimerkkejä sovellusten integroimiseen.

Ennen yksityiskohtiin menemistä tarkastellaan perus cURL-pyyntöä:

```php
// Luo ja alusta cURL-istunto
$curl = curl_init();

// Aseta URL
curl_setopt($curl, CURLOPT_URL, "api.example.com");

// Palauta siirto merkkijonona
curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);

// Suorita cURL-istunto
$output = curl_exec($curl);

// Sulje cURL-istunto
curl_close($curl);
```

Huomaa `curl_exec()`-funktion käyttö cURL-istunnon suorittamiseen, ja saatu vastaus tallennetaan `$output`-muuttujaan, vaikka yhteys suljettaisiin `curl_close()`-funktiolla.

Nyt, kun ymmärrämme perusteet, kapseloidaan tämä uudelleenkäytettävään funktioon.

## PHP cURL -asetus selkeäksi

Kun ulkoisen API:n integrointiin liittyy useita kutsuja projektin eri osista, modulaarinen lähestymistapa on hyödyllinen. Tässä on PHP-skripti, joka määrittelee funktion cURL-pyyntöjen suorittamiseen parametrien joukolla.

```php
function callAPI($method, $url, $data, $headers = false) {
    $curl = curl_init();

    switch ($method) {
        case "POST":
            curl_setopt($curl, CURLOPT_POST, 1);
            if ($data) {
                curl_setopt($curl, CURLOPT_POSTFIELDS, $data);
            }
            break;
        case "PUT":
            curl_setopt($curl, CURLOPT_CUSTOMREQUEST, "PUT");
            if ($data) {
                curl_setopt($curl, CURLOPT_POSTFIELDS, $data);
            }
            break;
        default:
            if ($data) {
                $url = sprintf("%s?%s", $url, http_build_query($data));
            }
    }

    // Aseta cURL-optiot
    curl_setopt($curl, CURLOPT_URL, $url);
    curl_setopt($curl, CURLOPT_HTTPHEADER, $headers ? $headers : [
        'APIKEY: xxxxxxxxxxxx',
        'Content-Type: application/json',
    ]);
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($curl, CURLOPT_HTTPAUTH, CURLAUTH_BASIC);

    // Suorita cURL
    $result = curl_exec($curl);

    // Käsittele yhteysvirhe
    if(!$result) {
        die("Yhteyden epäonnistuminen");
    }

    // Sulje cURL-istunto
    curl_close($curl);

    return $result;
}
```

Tämä asetus mahdollistaa cURL-kutsujen suorittamisen ja käyttää `switch`-lauseketta erottamaan POST-, PUT- ja muut tyyppiset pyynnöt.

## PHP cURL -GET-pyyntö

Perus GET-pyyntöön liittyy kolme parametria: `$method`, `$url` ja `$data`. cURL-GET:ssä aseta `$data` arvoon `false`, koska GET-pyyntöihin ei liity dataa.

```php
$get_data = callAPI('GET', 'https://api.example.com/get_url/'.$user['User']['customer_id'], false);
$response = json_decode($get_data, true);
$errors = $response['response']['errors'];
$data = $response['response']['data'][0];
```

## PHP cURL -POST-pyyntö

POST-pyyntöihin tarvitaan dataa. Varmista, että JSON-data on oikein välttääksesi virheitä.

```php
$data_array = [
    "customer" => $user['User']['customer_id'],
    "payment" => [
        "number" => $this->request->data['account'],
        "routing" => $this->request->data['routing'],
        "method" => $this->request->data['method']
    ],
];

$make_call = callAPI('POST', 'https://api.example.com/post_url/', json_encode($data_array));
$response = json_decode($make_call, true);
$errors = $response['response']['errors'];
$data = $response['response']['data'][0];
```

## PHP cURL -PUT-pyyntö

PUT-pyyntöjä vastaavat lähes POST-pyyntöihin. Huomaa muutokset `callAPI()`-funktiossa PUT-pyyntöjen käsittelyyn.

```php
$data_array = [
    "amount" => (string) ($lease['amount'] / $tenant_count),
];

$update_plan = callAPI('PUT', 'https://api.example.com/put_url/'.$lease['plan_id'], json_encode($data_array));
$response = json_decode($update_plan, true);
$errors = $response['response']['errors'];
$data = $response['response']['data'][0];
```

## PHP cURL -DELETE-pyyntö

DELETE-pyyntöjä on yksinkertaista. Anna API-URL ja haluttu `$id` poistoa varten.

```php
callAPI('DELETE', 'https://api.example.com/delete_url/' . $id, false);
```

## Joustavat otsikot

Joustavien otsikoiden mahdollistamiseksi muokkaa `callAPI()`-funktiota hyväksymään mukautetut otsikot.

```php
function callAPI($method, $url, $data, $headers = false) {
    $curl = curl_init();

    switch ($method) {
        // ... (POST- ja PUT-tapaukset)
    }

    // Aseta cURL-optiot
    curl_setopt($curl, CURLOPT_URL, $url);

    // Mukauta otsikot, jos ne on annettu
    if (!$headers) {
        curl_setopt($curl, CURLOPT_HTTPHEADER, [
            'APIKEY: xxxxxxxx',
            'Content-Type: application/json',
        ]);
    } else {
        curl_setopt($curl, CURLOPT_HTTPHEADER, array_merge([
            'APIKEY: xxxxxxxxxx',
            'Content-Type: application/json',
        ], $headers));
    }

    curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($curl, CURLOPT_HTTPAUTH, CURLAUTH_BASIC);

    // Suorita cURL
    $result = curl_exec($curl);

    // Käsittele yhteysvirhe
    if (!$result) {
        die("Yhteyden epäonnistuminen");
    }

    // Sulje cURL-istunto
    curl_close($curl);

    return $result;
}
```

Tämä muutos esittelee lisäparametrin `$headers` funktioon. Jos mukautettuja otsikoita ei anneta, käytetään oletusotsikoita.

## Mukautetut otsikot -esimerkki

Osoittaaksesi mukautettujen otsikoiden käyttöä, harkitse seuraavaa esimerkkiä hakuparametrien sisällyttämisestä:

```php
$one_month_ago = date("Y-m-d", strtotime(date("Y-m-d", strtotime(date("Y-m-d"))) . "-1 month"));
$rent_header = 'Search: and[][created][greater]=' . $one_month_ago . '%and[][created][less]=' . date('Y-m-d') . '%';

// Tee API-kutsu mukautetulla hakusyötteellä
$make_call = callAPI('GET', 'https://api.example.com/get_url/', false, $rent_header);
```

Tässä tapauksessa `$rent_header` lisätään oletusotsikoihin mahdollistaen mukautetun hakulausekkeen lisäämisen.

Nämä esimerkit pyrkivät selventämään cURL API-kutsujen integrointiprosessia PHP:ssa, käsitellen erilaisia HTTP-metodeja ja skenaarioita. Mukauta näitä pohjia projektisi erityistarpeisiin.

# **Efficient cURL API Integration in PHP: Handling RESTful Operations with JSON Data** (English)

In a recent project, I encountered the task of seamlessly integrating an external API using cURL requests in PHP. Navigating through this process for the first time, I faced challenges in grasping the intricacies. To streamline my future endeavors and potentially assist others, I'm documenting my cURL API calls in this post.

The examples and functions provided here are compatible with PHP version 5.6+.

## **Understanding PHP cURL Basics**

cURL, short for 'Client URL Library,' empowers you to establish connections and communicate with diverse servers utilizing various protocols (HTTP, HTTPS, FTP, proxy, cookies, etc.). To delve deeper into the mechanics of cURL, refer to the [official PHP documentation](https://www.php.net/manual/en/intro.curl.php). This article aims to furnish more comprehensive examples for integrating applications.

Before diving into the specifics, let's examine a basic cURL request:

```php
// Create and initialize a cURL session
$curl = curl_init();

// Set the URL
curl_setopt($curl, CURLOPT_URL, "api.example.com");

// Return the transfer as a string
curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);

// Execute the cURL session
$output = curl_exec($curl);

// Close the cURL session
curl_close($curl);
```

Note the utilization of `curl_exec()` to execute the cURL session, and the obtained response is stored in the `$output` variable, even after closing the connection with `curl_close()`.

Now that we comprehend the fundamentals, let's encapsulate this into a reusable function.

## **Streamlined PHP cURL Setup**

Considering that integrating an external API involves multiple calls from different sections of your project, a modular approach is beneficial. Below is a PHP script that defines a function allowing the execution of cURL requests with a set of parameters.

```php
function callAPI($method, $url, $data, $headers = false) {
    $curl = curl_init();

    switch ($method) {
        case "POST":
            curl_setopt($curl, CURLOPT_POST, 1);
            if ($data) {
                curl_setopt($curl, CURLOPT_POSTFIELDS, $data);
            }
            break;
        case "PUT":
            curl_setopt($curl, CURLOPT_CUSTOMREQUEST, "PUT");
            if ($data) {
                curl_setopt($curl, CURLOPT_POSTFIELDS, $data);
            }
            break;
        default:
            if ($data) {
                $url = sprintf("%s?%s", $url, http_build_query($data));
            }
    }

    // Set cURL options
    curl_setopt($curl, CURLOPT_URL, $url);
    curl_setopt($curl, CURLOPT_HTTPHEADER, $headers ? $headers : [
        'APIKEY: xxxxxxxxxxxxxxxxxxx',
        'Content-Type: application/json',
    ]);
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($curl, CURLOPT_HTTPAUTH, CURLAUTH_BASIC);

    // Execute cURL
    $result = curl_exec($curl);

    // Handle connection failure
    if(!$result) {
        die("Connection Failure");
    }

    // Close cURL session
    curl_close($curl);

    return $result;
}
```

This setup allows the execution of cURL calls and utilizes a `switch` statement to differentiate between POST, PUT, and other types of requests.

## **PHP cURL GET Request**

A basic GET call involves three parameters: `$method`, `$url`, and `$data`. For a cURL GET, set `$data` to `false` as GET calls do not include data.

```php
$get_data = callAPI('GET', 'https://api.example.com/get_url/'.$user['User']['customer_id'], false);
$response = json_decode($get_data, true);
$errors = $response['response']['errors'];
$data = $response['response']['data'][0];
```

## **PHP cURL POST Request**

POST requests necessitate data. Ensure your JSON data is accurate to avoid errors.

```php
$data_array = [
    "customer" => $user['User']['customer_id'],
    "payment" => [
        "number" => $this->request->data['account'],
        "routing" => $this->request->data['routing'],
        "method" => $this->request->data['method']
    ],
];

$make_call = callAPI('POST', 'https://api.example.com/post_url/', json_encode($data_array));
$response = json_decode($make_call, true);
$errors = $response['response']['errors'];
$data = $response['response']['data'][0];
```

## **PHP cURL PUT Request**

PUT requests are similar to POST requests. Note the adjustments in the `callAPI()` function for handling PUT requests.

```php
$data_array = [
    "amount" => (string) ($lease['amount'] / $tenant_count),
];

$update_plan = callAPI('PUT', 'https://api.example.com/put_url/'.$lease['plan_id'], json_encode($data_array));
$response = json_decode($update_plan, true);
$errors = $response['response']['errors'];
$data = $response['response']['data'][0];
```

## **PHP cURL DELETE Request**

DELETE requests are straightforward. Simply provide the API URL with the desired `$id` for removal.

```php
callAPI('DELETE', 'https://api.example.com/delete_url/' . $id, false);
```

## **Flexible Headers Handling**

To accommodate flexible headers, modify the `callAPI()` function to accept custom headers.

```php
function callAPI($method, $url, $data, $headers = false) {
    $curl = curl_init();

    switch ($method) {
        // ... (POST and PUT cases)
    }

    // Set cURL options
    curl_setopt($curl, CURLOPT_URL, $url);

    // Customize headers if provided
    if (!$headers) {
        curl_setopt($curl, CURLOPT_HTTPHEADER, [
            'APIKEY: xxxxxxxxxxxxxxxxxxx',
            'Content-Type: application/json',
        ]);
    } else {
        curl_setopt($curl, CURLOPT_HTTPHEADER, array_merge([
            'APIKEY: xxxxxxxxxxxxxxxxxxx',
            'Content-Type: application/json',
        ], $headers));
    }

    curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($curl, CURLOPT_HTTPAUTH, CURLAUTH_BASIC);

    // Execute cURL
    $result = curl_exec($curl);

    // Handle connection failure
    if (!$result) {
        die("Connection Failure");
    }

    // Close cURL session
    curl_close($curl);

    return $result;
}
```

This modification introduces an additional `$headers` parameter to the function. If custom headers are not provided, the default headers are utilized.

## **Custom Headers Example**

To demonstrate custom headers, consider the following example incorporating search parameters:

```php
$one_month_ago = date("Y-m-d", strtotime(date("Y-m-d", strtotime(date("Y-m-d"))) . "-1 month"));
$rent_header = 'Search: and[][created][greater]=' . $one_month_ago . '%and[][created][less]=' . date('Y-m-d') . '%';

// Make the API call with custom search header
$make_call = callAPI('GET', 'https://api.example.com/get_url/', false, $rent_header);
```

In this instance, the `$rent_header` variable is appended to the default headers, enabling the addition of custom search queries.

These examples aim to demystify the process of integrating cURL API calls in PHP, covering various HTTP methods and scenarios. Customize these templates to suit your project's specific requirements.
