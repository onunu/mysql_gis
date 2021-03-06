# MySQLでGISデータを扱う
こんにちは。onunuです。本記事はMySQL casual AdventCalendar 17日目の記事です。  
本記事ではMySQL8で大幅に強化されたGISデータ周りのtipsについて触れたいと思います。ある程度尽くされた話題ではありますが、主にアップデートされた機能について言及していこうと思います。みなさまの快適なMySQLライフの一助になれば幸いです。

# GISデータとMySQL
## MySQLにおけるGISデータ
GIS(Geographic Information System)とは地理情報や空間に関するデータをコンピュータ上で取り扱うためのシステムをさします。MySQLはOGC(Open Geospatial Consortium)の策定する仕様に準拠していていて、具体的には以下のようなデータを取り扱うことができます。

- 緯度/経度の単一の組み合わせ(`POINT`)
- 緯度/経度の連なる組み合わせで、線分を示す(`LINESTRING`)
- 緯度/経度の連なる組み合わせで、領域を示す(`POLYGON`)

上記3つのいずれにも対応する`GEOMETRY`型も存在し、これは3つのクラスの親クラスのように振る舞います。
また、本記事では取り扱いませんが、それぞれのコレクションを保存するための、`MULTIPOINT`, `MULTILINESTRING`, `MULTIPOLYGON`, `GEOMETRYCOLLECTION` といったデータも存在します。

これらGISデータはMySQLにおいてはバイナリで保存されており、CLIから人間にわかる形では表示されません。後述する適切な関数を用いることで必要なデータを抽出する必要があります。

## ざっくりとした対応の歴史
- 4.1: MyISAMでGIS機能が利用できるようになった
  - GEOMETRYデータ型
  - Spartialインデックス
  - Spartial関数
- 5.0: InnoDBでもGIS機能が利用できるようになった
  - しかしSpartialインデックスは使えず
- 5.7: オープンソースのライブラリを利用した再実装がなされる
  - InnoDBで以下が利用可能に
    - GEOMETRYデータ型
    - Spartialインデックス
    - Spartial関数
    - GeoHash
    - GeoJSON
  - 挙動が紛らわしい関数(ex: `CONTAINS()`)などの非推奨化
- 8.0: OpenGIS標準準拠
  - 取り扱う上での演算、データ変換に役立つ関数の追加
    - `ST_X(geom, x)`
    - `ST_SRID(geom, srid)`
  - 5.7で非推奨になった関数の廃止
  - Geograpy
  - Spartial Data, Index, 関数のSRIDサポート

5.7まではInnoDB対応したと言っても座標平面の単純なデータの扱いしかできず、精度に不安がありました。8ではメルトカル図法や正距方位図法が扱えるようになり、より正確なデータの扱いができるようになりました。
SRIDとは、空間参照系の識別コードを表す整数値のことです。GISデータを取り扱う際においては重要なファクタではありますが、非常に込み入った話になりますので本記事では省略します。以下の記事が詳細に解説していますので、実際にGIS機能を利用する際にはどのSRIDが適切か参考にすると良いと思います。

[空間参照系の概要](https://qiita.com/boiledorange73/items/b98d3d1ef3abf7299aba)

# MySQLでGISデータを使ってみる
では実際にGISデータを使ってみます。サンプルデータとして、江戸の五色不動を使います。

| id | 名称 | 緯度経度 | GeoHash値 |
|:---|:----|:--------|:---------|
| 1 | 目黒不動 | 35.628611,139.708056  | xn76ejfxz6nn |
| 2 | 目白不動 | 35.716100,139.712460  | xn775jzt278g |
| 3 | 目赤不動 | 35.726296,139.7507333 | xn77hpg2u072 |
| 4 | 目青不動 | 35.6447853,139.665668 | xn76f0vmvvw0 |
| 5 | 目黄不動 | 35.644765,139.5978171 | xn76b8ujbcuc |

MySQLでポイントデータを扱う際は、実際にはGeoHash値として保存しておくと便利です。
これらのデータをMySQLにロードします。

```
mysql> CREATE TABLE five_colors (id BIGINT(20), name VARCHAR(100), latlon POINT, geohash VARCHAR(30), PRIMARY KEY(id)) ENGINE=InnoDB CHARSET=utf8;

mysql> INSERT INTO five_colors_new VALUES
(1, "目黒不動", ST_GeomFromText('POINT(35.628611 139.708056)', 4326),  "xn76ejfxz6nn"),
(2, "目白不動", ST_GeomFromText('POINT(35.716100 139.712460)', 4326),  "xn775jzt278g"),
(3, "目赤不動", ST_GeomFromText('POINT(35.726296 139.7507333)', 4326), "xn77hpg2u072"),
(4, "目青不動", ST_GeomFromText('POINT(35.6447853 139.665668)', 4326), "xn76f0vmvvw0"),
(5, "目黄不動", ST_GeomFromText('POINT(35.644765 139.5978171)', 4326), "xn76b8ujbcuc");
```
早速それっぽいのが出てきました。`POINT`型を含むGeometry系のデータは先述の通りバイナリなので、INSERTを行う際にも変換が必要です。同時に取り出す時も関数を使う必要があります。

```
mysql> SELECT ST_AsText(latlon) FROM five_colors;
+------------------------------+
| ST_AsText(latlon)            |
+------------------------------+
| POINT(35.628611 139.708056)  |
| POINT(35.7161 139.71246)     |
| POINT(35.726296 139.7507333) |
| POINT(35.6447853 139.665668) |
| POINT(35.644765 139.5978171) |
+------------------------------+
5 rows in set (0.00 sec)
```

## GeometryとGeoHash,その他の型への変換
先述したように、実際にMySQLでGISデータを取り扱う場合、POINTに限って言えばGeoHash値として扱うのが便利です。  
GeoHashは緯度経度を元にハッシュ化した値で、テキストデータとして取り扱いが楽なだけでなく、文字列の近さが実際の距離の近さに対応していること、他システムでも同様に扱える点で有用です。
また、GeoHashは桁数によって精度を担保しており、10桁の場合、南北の距離が0.59m、東西の距離が0.97mになります。必要とされる精度によって桁数を調整すると良いでしょう。

MySQLの`POINT`型, 数値型との変換は、以下のように行います。
### GeoHash -> 緯度経度を数値でとる
緯度、経度はそれぞれ `ST_LatFromGeoHash()`, `ST_LongFromGeoHash()`関数で取り出します

```
mysql> SELECT name, ST_LatFromGeoHash(geohash), ST_LongFromGeoHash(geohash) FROM five_colors WHERE id = 1;
+--------------+----------------------------+-----------------------------+
| name         | ST_LatFromGeoHash(geohash) | ST_LongFromGeoHash(geohash) |
+--------------+----------------------------+-----------------------------+
| 目黒不動     |                  35.628631 |                  139.705901 |
+--------------+----------------------------+-----------------------------+
1 row in set (0.00 sec)
```

### GeoHash -> POINT
GeoHash値をPOINT型に変換することももちろんできます
```
mysql> SELECT name, ST_AsText(ST_PointFromGeoHash(geohash, 0)) FROM five_colors WHERE id = 1;
+--------------+--------------------------------------------+
| name         | ST_AsText(ST_PointFromGeoHash(geohash, 0)) |
+--------------+--------------------------------------------+
| 目黒不動     | POINT(139.705901 35.628631)                |
+--------------+--------------------------------------------+
1 row in set (0.00 sec)
```
`ST_PointFromGeoHash()`の第2引数はSRIDを指定します。緯度経度を取るだけなら0で事足りるでしょう。

### 数値 -> GeoHash
緯度/経度からGeoHashへの変換は以下のように行います。第3引数でハッシュ化した際の桁数を指定します。
```
mysql> SELECT ST_GeoHash(139.705901, 35.628631, 12);
+---------------------------------------+
| ST_GeoHash(139.705901, 35.628631, 12) |
+---------------------------------------+
| xn76ejfxz6nn                          |
+---------------------------------------+
1 row in set (0.00 sec)
```

### POINT -> 数値
`POINT`型からの数値の変換には注意が必要です。DBが持つデータをそのまま数値として出したいだけであれば、`ST_X()`, `ST_Y()`関数を使うことで取得できます。しかしこれとは別に、`ST_Latitude()`,`ST_Longitude()`関数も用意されています。この差は前述したSRIDに関するもので、`ST_X()`, `ST_Y()`は単なる2次元座標系における座標を返しているのに対し、`ST_Latitude()`,`ST_Longitude()`はSRIDに基づいた緯度/経度を返します。そのためデータの定義時にSRIDを指定しない(SRID=0)場合、`ST_Latitude()`, `ST_Longitude()`は実行時にエラーになります。

サンプルデータはSRIDを指定していませんでした。
```
mysql> SELECT ST_X(latlon), ST_Y(latlon) FROM five_colors WHERE id = 1;
+--------------+--------------+
| ST_X(latlon) | ST_Y(latlon) |
+--------------+--------------+
|    35.628611 |   139.708056 |
+--------------+--------------+
1 row in set (0.00 sec)
```
`ST_Latitude()`,`ST_Longitude()`を実行した場合は以下のようになります。
```
mysql> SELECT ST_Latitude(latlon), ST_Longitude(latlon) FROM five_colors WHERE id = 1;
ERROR 3726 (22S00): Function st_latitude is only defined for geographic spatial reference systems, but one of its arguments is in SRID 0, which is not geographic.
```
エラーの内容をみると、`SRID 0, which is not geographic.`とあり、つまり地理空間データではないと言われていますね。  
`ST_Latitude()`, `ST_Longitude()`を正常に実行するには、データの定義時点でSRIDを指定します。
```
mysql> SET @meguro = ST_GeomFromText('POINT(35.628611 139.708056)', 4326);
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT ST_Latitude(@meguro), ST_Longitude(@meguro);
+----------------------+-----------------------+
| ST_Latitude(@meguro) | ST_Longitude(@meguro) |
+----------------------+-----------------------+
|            35.628611 |            139.708056 |
+----------------------+-----------------------+
1 row in set (0.00 sec)
```
SRID=4326は世界測地系1984(WGS84)を指します。

### 数値 -> Point
ここまで説明をせず書いていましたが、数値からGeometry型に変換することもできます。しかし、用意されているのは

- `ST_GeomFromText()`: Text -> Geometry
- `ST_GeomFromGeoJSON()`: GeoJSON -> Geometry
- `ST_GeomFromWKB()`: WKB(WellKnowTextをBinary化したもの) -> Geometry

の3種類ですが、バイナリであるWKBを人間が記述したり入力するのは難しいのでよく使う文脈でいえば上の2種類を多用することになりそうです。GeoJSONに関する詳しい説明は省きますが、以下のように利用します。

```
mysql> SELECT ST_AsText(ST_GeomFromText('POINT(35.628611 139.708056)', 0));
+--------------------------------------------------------------+
| ST_AsText(ST_GeomFromText('POINT(35.628611 139.708056)', 0)) |
+--------------------------------------------------------------+
| POINT(35.628611 139.708056)                                  |
+--------------------------------------------------------------+
1 row in set (0.00 sec)
```

```
mysql> SET @json = '
  {
    "type":"Point",
    "coordinates": [139.708056, 35.628611],
    "crs": {
      "type":"name",
        "properties": {
          "name":"EPSG:4326"
        }
    }
  }
';
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT ST_AsText(ST_GeomFromGeoJSON(@json));
+--------------------------------------+
| ST_AsText(ST_GeomFromGeoJSON(@json)) |
+--------------------------------------+
| POINT(35.628611 139.708056)          |
+--------------------------------------+
1 row in set (0.00 sec)
```
`ST_GeomFromText()`は第2引数にSRIDを指定できます(デフォルトは0)が、`ST_GeomFromGeoJSON()`はGeoJson中にSRIDを持たせる必要がある点に注意してください。

## GISデータを有効活用する
GISデータを有効活用するための関数についていくつか説明します。
### 2点間の距離を取る
2点間の距離を取るには、`ST_Distance()`関数を使います。
```
mysql> SELECT p.name AS start, s.name AS end,  ST_Distance(p.latlon, s.latlon) FROM five_colors AS p CROSS JOIN five_colors AS s WHERE p.id < s.id;
+--------------+--------------+---------------------------------+
| start        | end          | ST_Distance(p.latlon, s.latlon) |
+--------------+--------------+---------------------------------+
| 目黒不動      | 目白不動       |             0.08759977361271878 |
| 目黒不動      | 目赤不動       |             0.10660070900463282 |
| 目黒不動      | 目青不動       |            0.045369048088857965 |
| 目黒不動      | 目黄不動       |             0.11141618728538899 |
| 目白不動      | 目赤不動       |             0.03960812932834764 |
| 目白不動      | 目青不動       |              0.0852952384373689 |
| 目白不動      | 目黄不動       |             0.13502472642227886 |
| 目赤不動      | 目青不動       |             0.11781383398641554 |
| 目赤不動      | 目黄不動       |             0.17329358956247118 |
| 目青不動      | 目黄不動       |             0.06785090303675664 |
+--------------+--------------+---------------------------------+
10 rows in set (0.00 sec)
```

単位が謎ですよね。これはSRIDが未指定(0)のためにユークリッド距離を返しているからです。現実世界に準じて球面距離を返して欲しいので、SRIDを4326に指定してデータを再構築してみます。

```
mysql> CREATE TABLE five_colors_new (id BIGINT(20), name VARCHAR(100), latlon POINT, geohash VARCHAR(30), PRIMARY KEY(id)) ENGINE=InnoDB CHARSET=utf8;
Query OK, 0 rows affected, 1 warning (0.09 sec)
mysql> INSERT INTO five_colors_new VALUES
    -> (1, "目黒不動", ST_GeomFromText('POINT(35.628611 139.708056)', 4326),  "xn76ejfxz6nn"),
    -> (2, "目白不動", ST_GeomFromText('POINT(35.716100 139.712460)', 4326),  "xn775jzt278g"),
    -> (3, "目赤不動", ST_GeomFromText('POINT(35.726296 139.7507333)', 4326), "xn77hpg2u072"),
    -> (4, "目青不動", ST_GeomFromText('POINT(35.6447853 139.665668)', 4326), "xn76f0vmvvw0"),
    -> (5, "目黄不動", ST_GeomFromText('POINT(35.644765 139.5978171)', 4326), "xn76b8ujbcuc");
Query OK, 5 rows affected (0.08 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> SELECT p.name AS start, s.name AS end, ST_Distance(p.latlon, s.latlon) FROM five_colors_new AS p CROSS JOIN five_colors_new AS s WHERE p.id < s.id;
+--------------+--------------+---------------------------------+
| start        | end          | ST_Distance(p.latlon, s.latlon) |
+--------------+--------------+---------------------------------+
| 目黒不動      | 目白不動       |               9715.421270161714 |
| 目黒不動      | 目赤不動       |              11506.541551462107 |
| 目黒不動      | 目青不動       |               4238.018883326555 |
| 目黒不動      | 目黄不動       |              10144.522112266772 |
| 目白不動      | 目赤不動       |               3643.068109360428 |
| 目白不動      | 目青不動       |               8975.104665872295 |
| 目白不動      | 目黄不動       |              13051.864958700196 |
| 目赤不動      | 目青不動       |              11877.888501149639 |
| 目赤不動      | 目黄不動       |              16535.834616701708 |
| 目青不動      | 目黄不動       |               6145.005204768069 |
+--------------+--------------+---------------------------------+
10 rows in set (0.00 sec)
```

ご覧のように数字が大きく変わりました。単位はmになっています。

### 領域を表現する
GISデータで領域を表現するには、`POLYGON`型を使います。
サンプルデータとして、代々木公園はおよそ以下のGeoJSONで表現できます。

```json
{
  "type": "Polygon",
  "coordinates": [
    [
      [139.70324277877808,35.680184793238894],
      [139.70096826553345,35.67988848691441],
      [139.69813585281372,35.680899409847584],
      [139.6959900856018,35.67985362727446],
      [139.69446659088135,35.679539889],
      [139.69343662261963,35.676803349816815],
      [139.69204187393188,35.67483368013732],
      [139.69135522842407,35.672445608339494],
      [139.6909260749817,35.66988314526595],
      [139.69274997711182,35.66674532334593],
      [139.69356536865234,35.66634437054716],
      [139.6940803527832,35.66583881849588],
      [139.6977710723877,35.66725086965674],
      [139.69757795333862,35.66834911440362],
      [139.70030307769775,35.668593166739974],
      [139.7023630142212,35.66951707239733],
      [139.70369338989258,35.67443276796514],
      [139.70401525497437,35.67603640457032],
      [139.70324277877808,35.680184793238894]
    ]
  ],
  "crs": {
    "type":"name",
      "properties": {
        "name":"EPSG:4326"
      }
  }
}
```

GeometryとしてMySQLに読み込むには、

```
mysql> SET @json = '{{ 上のJSON}}';
Query OK, 0 rows affected (0.00 sec)

mysql> SET @yoyogi_park = ST_GeomFromGeoJSON(@json);
Query OK, 0 rows affected (0.00 sec)
```

とすればOKです。

### 領域に他の点や領域が含まれているかを判断する
`ST_Contains()`を使うと領域、点が(完全に)含まれているかどうかを検証できます。
第1引数の中に第2引数の指定する領域または座標が含まれるかどうかを0,1で返します。

```
mysql> SELECT ST_Contains(@yoyogi_park, latlon) FROM five_colors_new;
+-----------------------------------+
| ST_Contains(@yoyogi_park, latlon) |
+-----------------------------------+
|                                 0 |
|                                 0 |
|                                 0 |
|                                 0 |
|                                 0 |
+-----------------------------------+
5 rows in set (0.00 sec)
```
```
mysql> SET @meiji_shrine = ST_GeomFromText('POINT(35.6763976 139.6971372)', 4326);
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT ST_Contains(@yoyogi_park, @meiji_shrine);
+------------------------------------------+
| ST_Contains(@yoyogi_park, @meiji_shrine) |
+------------------------------------------+
|                                        1 |
+------------------------------------------+
1 row in set (0.00 sec)
```

他にも`ST_Centroid()`、`ST_Crosses()`など様々な便利な関数が用意されています。
5.7よりも前と8では関数の種類、数に大きな違いがありますので、GIS機能を利用する際には必ず最新のリファレンスを参照するようにしてください。

# 五色不動巡りを最適化する
要は巡回サラリーマン問題です。不動尊は5つあるので、`5! = 120` 通りの経路が存在します(順序を真逆にしたものがあるので有効なのはこのさらに半分)。

かっこよく書きたかったのですが、ちょっとワンクエリでやるのは無理でした。結局脳筋な解き方をしています。
あと今更ですが、江戸の五色不動といった場合、目黄不動は最勝寺と永久寺の2箇所をさすそうです。今回は最勝寺だけを目黄不動として扱い、問題を簡略化していますので、ご了承ください。

```
WITH t AS (
  SELECT
    f.name AS first,
    s.name AS second,
    t.name AS third,
    fo.name AS fourth,
    fi.name AS fifth,
    ST_Distance(f.latlon, s.latlon) + ST_Distance(s.latlon, t.latlon)
    + ST_Distance(t.latlon, fo.latlon) + ST_Distance(fo.latlon, fi.latlon) AS distance
  FROM
    five_colors_new AS f
    CROSS JOIN five_colors_new AS s
    CROSS JOIN five_colors_new AS t
    CROSS JOIN five_colors_new AS fo
    CROSS JOIN five_colors_new AS fi
  WHERE
    f.id != s.id
    AND f.id != t.id
    AND f.id != fo.id
    AND f.id != fi.id
    AND s.id != t.id
    AND s.id != fo.id
    AND s.id != fi.id
    AND t.id != fo.id
    AND t.id != fi.id
    AND fo.id != fi.id
)
SELECT * FROM t ORDER BY distance LIMIT 2
;
```
返ってきた結果が以下のようになります。

```
+--------------+--------------+--------------+--------------+--------------+--------------------+
| first        | second       | third        | fourth       | fifth        | distance           |
+--------------+--------------+--------------+--------------+--------------+--------------------+
| 目黄不動     | 目青不動     | 目黒不動     | 目白不動     | 目赤不動     | 23741.513467616765 |
| 目赤不動     | 目白不動     | 目黒不動     | 目青不動     | 目黄不動     |  23741.51346761677 |
+--------------+--------------+--------------+--------------+--------------+--------------------+
2 rows in set (0.01 sec)
```

どうやら江戸の五色不動巡りをする場合、目黄不動から始め、目青不動、目黒不動、目白不動、目赤不動と回るか、その逆が効率がよいようです。参考になれば幸いです。あと当然ですが、交通手段とかは考慮していませんので、悪しからず。

# おわりに
MySQLの強化されたGIS機能に関してざっくりと解説しました。今回説明できたことはごくわずかで、便利かつ高精度な関数、機能が他にも多数あります。もし興味をお持ちであれば公式のリファレンスを参照されることを強くおすすめします。拙い説明でしたが読んでくださりありがとうございました。
