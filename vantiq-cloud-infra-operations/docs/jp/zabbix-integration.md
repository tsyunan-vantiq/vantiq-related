# Zabbixとのインテグレーション調査

- Zabbix Web, Server, Agent構築：https://github.com/zabbix/zabbix-docker

# Zabbix ServerからAgent Item　データをVANTIQに送信するできるかを検証

## Zabbix Connector

- https://www.zabbix.com/documentation/6.4/jp/manual/config/export/streaming

- 送信先：https://git.zabbix.com/projects/ZT
  - [Kafka broker](https://git.zabbix.com/projects/ZT/repos/kafka-connector/browse)　公式Kafka connector
  - [Http server](https://git.zabbix.com/projects/ZT/repos/receiver/browse)　サンプルGolangアプリ、イベントとhistory データをファイルに書き込む

VANTIQ topicに送信することができるかを検証：
- http://zabbix-web.vantiqjp.com/zabbix.php?action=connector.list

![alt text](image.png)

送信できないです、なぜできないのが分からない
![alt text](image-1.png)

## Zabbix Alerts Webhook

既存Media Typeがないため、作成する
![alt text](image-2.png)
- http://zabbix-web.vantiqjp.com/zabbix.php?action=mediatype.edit&mediatypeid=42
- ![alt text](image-3.png)

Webhookスクリプト：
```js
try {
    Zabbix.log(4, '[ VANTIQ webhook ] Started with params: ' + value);

    var result = {
        'tags': {
            'endpoint': 'vantiq'
        }
    },
        // value値はGlobal値らしくて、paramsを取得することができる
        params = JSON.parse(value),
        req = new HttpRequest(),
        fields = {},
        resp;
    // 
    if (params.HTTPProxy) {
        req.setProxy(params.HTTPProxy);
    }

    req.addHeader('Content-Type: application/json');
    req.addHeader('Authorization: Bearer ' + params.authentication);

    // fields.summary = params.summary;
    // fields.description = params.description;
    // fields.project = { key: params.project_key };
    // fields.issuetype = { id: params.issue_id };

    resp = req.post('https://internal.vantiq.com/api/v1/resources/topics/zabbixtopic',
        JSON.stringify({ "fields": value })
    );

    if (req.getStatus() != 201) {
        throw 'Response code: ' + req.getStatus();
    }

    resp = JSON.parse(resp);
    // result.tags.issue_id = resp.id;
    // result.tags.issue_key = resp.key;

    return JSON.stringify(result);
}
catch (error) {
    Zabbix.log(4, '[ VANTIQ webhook ] Event Post failed json : ' + JSON.stringify({ "fields": fields }));
    Zabbix.log(3, '[ VANTIQ webhook ] Event Post failed : ' + error);

    throw 'Failed with error: ' + error;
}
```

送信をテストする、送信はできるです。がAgent Itemデータを送信するには使える[Agent Itemデータ関連マクロ](https://www.zabbix.com/documentation/6.4/jp/manual/appendix/macros/supported_by_location)がなさそう、スクリプトの実装は厳しい

使えそうなマクロ：{ITEM.VALUE}
- https://www.zabbix.com/documentation/6.4/jp/manual/appendix/macros/supported_by_location

![alt text](image-5.png)
![alt text](image-6.png)

### まとめ：

アラートWebHookはアラートイベント、障害イベントを通知するために使われている。Agent Itemデータのストミーングとの関係が見えない
