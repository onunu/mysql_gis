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
