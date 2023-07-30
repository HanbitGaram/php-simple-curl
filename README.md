# The Simple CURL Code

```php
<?php
function httpRequest($method='GET', $args=[]){
        if(!isset($args['ua']))
            $args['ua'] = "Mozilla/5.0 Application";

        $method = strtoupper($method);

        $args = [
            'url'=>'',
            'headers'=>[],
            'ua'=>'',
            'accept'=>'text/html, application/xhtml+xml, application/xml;q=0.9, */*;q=0.8',
            'timeout'=>10,
            'json'=>true,
            'follow'=>true,
            'showHeader'=>true,
            'followCount'=>5,
            'verifyPeer'=>false,
            'data'=>[],
            'unauthorized'=>false,
            ...$args
        ];

        $args['headers'][] = "User-Agent: ".trim($args['ua']);
        if($args['json']!==true)
            $args['headers'][] = 'Accept: '.trim($args['accept']);
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $args['url']);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, $args['timeout']);
        curl_setopt($ch, CURLOPT_HEADER, $args['showHeader']);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $args['headers']);
        curl_setopt($ch, CURLOPT_FOLLOWLOCATION, $args['follow']);
        curl_setopt($ch, CURLOPT_MAXREDIRS, $args['followCount']);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, $args['verifyPeer']);
        //curl_setopt($ch, CURLOPT_RESOLVE, [
        //    "hanb.jp:443:127.0.0.1"
        //]);
        if($method==='POST'){
            curl_setopt($ch, CURLOPT_POST, true);
            if($args['json']!==true)
                curl_setopt($ch, CURLOPT_POSTFIELDS, $args['data']);
            else
                curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($args['data']));
        }
        $res = curl_exec($ch);
        $retry = 0;
        while(curl_errno($ch)!==0 && $retry < 3){
            $res = curl_exec($ch);
            $retry++;
        }

        if(curl_errno($ch)!==0) return false;
        curl_close($ch);

        $response = [
            'header'=>null,
            'body'=>null
        ];

        // 헤더 및 콘텐츠 분리
        $_ = explode("\r\n\r\n", $res);
        $_header = explode("\n", $_[0]);
        foreach($_header as $val){
            // 캐리지 리턴 제거
            $val = str_replace("\r", "", $val);
            $position = strpos($val, ':');

            // 헤더는 512바이트도 많이 받아오는 편임.
            $_name = strtoupper(trim(substr($val, 0, $position)));
            $_value = trim(substr($val, $position, 512));
            if($_name==="" && str_starts_with($_value, 'HTTP/'))
                $_name = "C-HTTP-PROTO";

            // 만약, HTTP 프로토콜 헤더가 아닌 경우
            if(str_starts_with($_value, ':'))
                $_value = trim(substr($_value, 1, 512));

            $response['header'][$_name] = $_value;
        }

        $response['body'] = $_[1];
        unset($_, $_header, $position, $_name, $_value);

        try{
            $json = json_decode($response['body'],true);
            if($json===null || $json===false) return $response;
            $response['body'] = $json;
            return $response;
        }catch(\Exception $e){
            return $response;
        }
    }
```
