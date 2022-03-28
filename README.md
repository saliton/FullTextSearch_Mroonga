[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/FullTextSearch_Mroonga/blob/main/FullTextSearch_Mroonga.ipynb)

# Colabで全文検索（その4：Mroonga編）

各種全文検索ツールをColabで動かしてみるシリーズです。全7回の予定です。今回はMroongaです。MroongaはGroongaをMySQLで使えるようにした拡張機能です。MySQLのネイティブな全文検索に対してどれぐらい優位性があるのでしょうか。

処理時間の計測はストレージのキャッシュとの兼ね合いがあるので、2回測ります。2回目は全てがメモリに載った状態での性能評価になります。ただ1回目もデータを投入した直後なので、メモリに載ってしまっている可能性があります。

## 準備

まずは検索対象のテキストを日本語wikiから取得して、Google Driveに保存します。（※ Google Driveに約１GBの空き容量が必要です。以前のデータが残っている場合は取得せず再利用します。）

Google Driveのマウント


```python
from google.colab import drive
drive.mount('/content/drive')
```

    Mounted at /content/drive


jawikiの取得とjson形式に変換。90分ほど時間がかかります。他の全文検索シリーズでも同じデータを使うので、他の記事も試す場合は wiki.json.bz2 を捨てずに残しておくことをおすすめします。


```shell
%%time
%cd /content/
import os
if not os.path.exists('/content/drive/MyDrive/wiki.json.bz2'):
    !wget https://dumps.wikimedia.org/jawiki/latest/jawiki-latest-pages-articles.xml.bz2
    !pip install wikiextractor
    !python -m wikiextractor.WikiExtractor --no-templates --processes 4 --json -b 10G -o - jawiki-latest-pages-articles.xml.bz2 | bzip2 -c > /content/drive/MyDrive/wiki.json.bz2
```

    /content
    CPU times: user 1.06 ms, sys: 964 µs, total: 2.02 ms
    Wall time: 8.16 ms


json形式に変換されたデータを確認


```python
import json
import bz2

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    for n, line in enumerate(fin):
        data = json.loads(line)
        print(data['title'].strip(), data['text'].replace('\n', '')[:40], sep='\t')
        if n == 5:
            break
```

    アンパサンド	アンパサンド（&amp;, ）は、並立助詞「…と…」を意味する記号である。ラテン
    言語	言語（げんご）は、広辞苑や大辞泉には次のように解説されている。『日本大百科事典』
    日本語	 日本語（にほんご、にっぽんご）は、日本国内や、かつての日本領だった国、そして日
    地理学	地理学（ちりがく、、、伊：geografia、）は、。地域や空間、場所、自然環境
    EU (曖昧さ回避)	EU
    国の一覧	国の一覧（くにのいちらん）は、世界の独立国の一覧。対象.国際法上国家と言えるか否


## Mroongaのインストール


```shell
!sudo apt-get install -y -V software-properties-common lsb-release
!sudo add-apt-repository -y universe
!sudo add-apt-repository "deb http://security.ubuntu.com/ubuntu $(lsb_release --short --codename)-security main restricted"
!sudo add-apt-repository -y ppa:groonga/ppa
!sudo apt-get update
!sudo apt-get install -y -V mysql-server-mroonga
```

    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    lsb-release is already the newest version (9.20170808ubuntu1).
    lsb-release set to manually installed.
    ・
    ・
    ・
    W: Target Packages (restricted/binary-all/Packages) is configured multiple times in /etc/apt/sources.list:44 and /etc/apt/sources.list:51


## MySQLの起動

サーバーを起動します。


```shell
!service mysql start
```

     * Starting MySQL database server mysqld
    No directory, logging in with HOME=/
       ...done.


Mroongaを導入します。


```shell
!mysql < /usr/share/mroonga/install.sql
```

Mroongaが導入されていることを確認します。


```shell
!echo "show engines;" | mysql
```

    Engine	Support	Comment	Transactions	XA	Savepoints
    InnoDB	DEFAULT	Supports transactions, row-level locking, and foreign keys	YES	YES	YES
    MRG_MYISAM	YES	Collection of identical MyISAM tables	NO	NO	NO
    MEMORY	YES	Hash based, stored in memory, useful for temporary tables	NO	NO	NO
    BLACKHOLE	YES	/dev/null storage engine (anything you write to it disappears)	NO	NO	NO
    MyISAM	YES	MyISAM storage engine	NO	NO	NO
    CSV	YES	CSV storage engine	NO	NO	NO
    Mroonga	YES	CJK-ready fulltext search, column store	NO	NO	NO
    PERFORMANCE_SCHEMA	YES	Performance Schema	NO	NO	NO
    FEDERATED	NO	Federated MySQL storage engine	NULL	NULL	NULL
    ARCHIVE	YES	Archive storage engine	NO	NO	NO


DBを作成します。


```shell
!echo "create database db" | mysql
```

## データのインポート

pythonからMySQLを操作できるmysqlclientをインストールします。


```shell
!pip install mysqlclient
```

    Collecting mysqlclient
      Downloading mysqlclient-2.1.0.tar.gz (87 kB)
    [K     |████████████████████████████████| 87 kB 6.0 MB/s 
    [?25hBuilding wheels for collected packages: mysqlclient
      Building wheel for mysqlclient (setup.py) ... [?25l[?25hdone
      Created wheel for mysqlclient: filename=mysqlclient-2.1.0-cp37-cp37m-linux_x86_64.whl size=99969 sha256=32eb347460dc8959efcca474f36e167fbdb8de730bc5818cf0c4e74e6ee4fec3
      Stored in directory: /root/.cache/pip/wheels/97/d4/df/08cd6e1fa4a8691b268ab254bd0fa589827ab5b65638c010b4
    Successfully built mysqlclient
    Installing collected packages: mysqlclient
    Successfully installed mysqlclient-2.1.0


データを解凍して50万件のデータをインポートします。全文検索用のインデックスを作りながら読み込むので、1時間以上かかります。


```python
import MySQLdb
import json
import bz2
from tqdm.notebook import tqdm

db = MySQLdb.connect(host='localhost', user='root', db='db', charset='utf8mb4')
cursor = db.cursor()

cursor.execute('drop table if exists wiki_jp')
cursor.execute('create table wiki_jp('
 'id bigint unsigned not null auto_increment primary key,'
 'title tinytext collate utf8mb4_unicode_ci storage memory,'
 'body mediumtext collate utf8mb4_unicode_ci storage memory,'
 'fulltext index (body)'
 ') engine = Mroonga default charset utf8')

limit = 500000
insert_wiki = 'insert into wiki_jp (title, body) values (%s, %s);'

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    n = 0
    for line in tqdm(fin, total=limit*1.5):
        data = json.loads(line)
        title = data['title'].strip()
        body = data['text'].replace('\n', '')
        if len(title) > 0 and len(body) > 0:
            cursor.execute(insert_wiki, (title, body))
            n += 1
        if n == limit:
            break
db.commit()
db.close()
```


      0%|          | 0/750000.0 [00:00<?, ?it/s]


データが書き込まれたテーブルを確認します。


```shell
!echo "show columns from db.wiki_jp" | mysql
```

    Field	Type	Null	Key	Default	Extra
    id	bigint(20) unsigned	NO	PRI	NULL	auto_increment
    title	tinytext	YES		NULL	
    body	mediumtext	YES	MUL	NULL	


登録件数を確認します。


```shell
!echo "select count(*) from db.wiki_jp" | mysql
```

    count(*)
    500000


## インデックスを使わない検索

bodyに「日本語」を含むレコードの数を取得します。


```shell
%%time
!echo "select sql_no_cache count(*) from db.wiki_jp where body like '%日本語%'" | mysql -vvv
```

    --------------
    select sql_no_cache count(*) from db.wiki_jp where body like '%日本語%'
    --------------
    
    +----------+
    | count(*) |
    +----------+
    |    17006 |
    +----------+
    1 row in set, 1 warning (7.84 sec)
    
    Bye
    CPU times: user 76.2 ms, sys: 13.7 ms, total: 89.8 ms
    Wall time: 7.96 s


bodyに「日本語」を含むレコードを取得します。


```shell
%%time
!echo "select sql_no_cache * from db.wiki_jp where body like '%日本語%'" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (8.91 sec)
    CPU times: user 112 ms, sys: 18.5 ms, total: 130 ms
    Wall time: 12.7 s


2回目


```shell
%%time
!echo "select sql_no_cache * from db.wiki_jp where body like '%日本語%'" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (8.83 sec)
    CPU times: user 119 ms, sys: 12.8 ms, total: 132 ms
    Wall time: 12.6 s


## インデックスを使った検索

bodyに「日本語」を含むレコードの数を取得します。


```shell
%%time
!echo "select sql_no_cache count(*) from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql -vvv
```

    --------------
    select sql_no_cache count(*) from db.wiki_jp where match (body) against ('日本語' in boolean mode)
    --------------
    
    +----------+
    | count(*) |
    +----------+
    |    17006 |
    +----------+
    1 row in set, 1 warning (0.08 sec)
    
    Bye
    CPU times: user 7.8 ms, sys: 3.96 ms, total: 11.8 ms
    Wall time: 118 ms


MySQLが2.5秒、Groongaが0.3秒ですから、相当速いです。なんらかの並列検索が働いている感じです。


bodyに「日本語」を含むレコードを取得します。


```shell
%%time
!echo "select sql_no_cache * from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (1.17 sec)
    CPU times: user 57 ms, sys: 11 ms, total: 68 ms
    Wall time: 5.03 s


2回目


```shell
%%time
!echo "select sql_no_cache * from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (1.17 sec)
    CPU times: user 47.5 ms, sys: 13.3 ms, total: 60.8 ms
    Wall time: 4.93 s


MySQL単体では8秒程度でしたから、5秒で少し速くなっています。ただ、Groonga単体では2秒程度でした。

参考として、idとtitleだけを取得してみます。


```shell
%%time
!echo "select sql_no_cache id title from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (0.14 sec)
    CPU times: user 10.8 ms, sys: 7.1 ms, total: 17.9 ms
    Wall time: 219 ms


やはりbodyの読み込みの時間が大半を占めているようです。

## MySQLの停止


```shell
!service mysql stop
```

     * Stopping MySQL database server mysqld
       ...done.

