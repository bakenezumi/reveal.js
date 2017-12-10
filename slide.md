Doma on Scala
------
Doma勉 2017



自己紹介
------
- スクィーラ
- 福岡のSier勤務
- 受託業務アプリ開発 / パッケージ開発
- 主にJava, 最近は若干Scala
- Doma歴5年くらい （S2Dao含めると10年強）



なぜScalaからDomaを使おうとしたのか
-----
- 現在チームではPlay-Java + Doma2がメイン構成<!-- .element: class="fragment" -->

⇒ Scalaに移行したい<!-- .element: class="fragment" -->

- 他メンバーにPlay, Scala, ＋他のなにかのOR-Mapperを全て覚えさせるのはハードルが高いが、使い慣れたDomaを使えるのであれば移行がスムーズになるのではと思った<!-- .element: class="fragment" -->



Scala＋Domaの話をする前に、
Pluggable Annotation Processing APIについて少し話します



Pluggable Annotation Processing API
-----

- Java標準のアノテーションマクロ<!-- .element: class="fragment" data-fragment-index="1"-->

- コンパイル時に<!-- .element: class="fragment" data-fragment-index="2"-->
  - アノテーションが付与されたコードの検証<!-- .element: class="fragment" data-fragment-index="2"-->
  - Javaソースコードの自動生成<!-- .element: class="fragment" data-fragment-index="2"-->

  が行える<!-- .element: class="fragment" data-fragment-index="2"-->



もたらされるもの
-----
- コンパイル時にコード検証
  - エラーメッセージによるコーディングのガイド<!-- .element: class="fragment" style="font-size:90%" -->
  - 実行時にエラーが先送りされない<!-- .element: class="fragment" style="font-size:90%" -->

- 自動生成
  - 事前にリフレクションを済ませられるため実行時リフレクションがいらない<!-- .element: class="fragment" style="font-size:90%" -->
  - ボイラープレートが減る<!-- .element: class="fragment" style="font-size:90%" -->



Doma2のAnnotation Processing
-----
EmpDao.java
```java
@Dao public interface EmpDao {
    @Select List<Emp> findAll();
}
```

↓

EmpDaoImpl.java
```java
public class EmpDaoImpl
  extends org.seasar.doma.internal.jdbc.dao.AbstractDao
  implements EmpDao { .../* Dao実装 */ }
```



Doma2のAnnotation Processing
-----
Emp.java
```java
@Entity public classs EmpDao {
    @Id int id;
}
```
↓

_Emp.java
```java
public final class _Emp
  extends org.seasar.doma.jdbc.entity.AbstractEntityType<Emp> {
    .../* Entityのヘルパー実装 */ }
```



Doma2のAnnotation Processingの流れ
-----

![javac](./doma-javac.png)

<span style="font-size:60%">http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html </span>



Pluggable Annotation Processing APIを踏まえた上でScalaでDomaを動かします



Step1
----- 
Doma関連クラスはJavaで作り、利用をScalaで<!-- .element: class="fragment" -->

Step2
-----
Doma関連クラスもScalaで作る<!-- .element: class="fragment" -->

Step3
-----
Domala<!-- .element: class="fragment" -->



Step1～3では下記のJavaアプリをScalaにします

```java
public class SampleApp0 {
  private static final EmpDao dao = new EmpDaoImpl();
  public static void main(String[] args) {
    AppConfig.singleton().getTransactionManager().required(() -> {
      // create table & insert
      dao.create();
      final List<Result<Emp>> inserted = Stream.of(
        new Emp(ID.of(-1), "scott", 10, -1),
        new Emp(ID.of(-1), "allen", 20, -1)
      ).map(dao::insert)
        .collect(Collectors.toList());
      System.out.println(inserted);

      // idが2のEmpのageを +1 にupdate
      final Optional<Result<Emp>> updated =
        dao
          .selectById(ID.of(2))
          .map(Emp::grawOld)
          .map(dao::update);
      System.out.println(updated);

      dao.selectAll().forEach(System.out::println);
      // =>
      //   Emp(id=ID(1), name=scott, age=10, version=1)
      //   Emp(id=ID(2), name=allen, age=21, version=2)
    });
  }}
```



Step1
----- 
Doma関連クラスはJavaで作り、利用をScalaで




Step1 project構成

```
project-root/
 - src/
    - main/
       - java/      #Doma関連クラスのソース
       - scala/     #Dao利用クラスのソース
       - resources/ #SQLファイル
 - target/
    - scala-2.12/
       - classes/   #ビルド結果の出力ディレクトリ
 - build.sbt
```



Step1 build.sbt（初版 要修正）
```scala
lazy val root = (project in file(".")).
  settings(
    inThisBuild(List(
      scalaVersion := "2.12.4",
      version      := "0.1.0-SNAPSHOT"
    )),
    name := "scala-doma-sample1",
    libraryDependencies ++= Seq(
      "org.seasar.doma" % "doma" % "2.19.0",
      "com.h2database" % "h2" % "1.4.193",
      scalaTest % Test
    )
  )
```



Step1 Entity (Java)

```java
@Entity(immutable = true)
public class Emp implements Serializable {
  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE)
  @SequenceGenerator(sequence = "emp_id_seq")
  final ID<Emp> id;
  final String name;
  final int age;
  @Version
  final int version;
  public Emp(ID<Emp> id, String name, int age, int version) {
    this.id = id;
    this.name = name;
    this.age = age;
    this.version = version;
  }
  /* 年齢 +1 */
  public Emp grawOld() {
    return new Emp(id, name, age + 1, version);
  }

  @Override
  public boolean equals(Object obj) {
    if(obj instanceof Emp) {
      Emp emp = ((Emp) obj);
      return emp.id.equals(id) && emp.name.equals(name) && emp.age == age && emp.version == version;
    } else {
      return false;
    }
  }

  @Override
  public String toString() {
    return (String.format("Emp(id=%s, name=%s, age=%d, version=%d)", id, name, age, version));
  }

  private static final long serialVersionUID = 1L;

}
```



Step1 Domain  (Java)

```java
@Domain(valueType = long.class, factoryMethod = "of")
public final class ID<ENTITY> implements Serializable {
  private final long value;

  private ID(final long value) {
    this.value = value;
  }

  public long getValue() {
    return value;
  }

  public static <ENTITY> ID<ENTITY> of(final long value) {
    return new ID<>(value);
  }

  @Override
  public boolean equals(Object obj) {
    return obj instanceof ID && ((ID) obj).value == value;
  }

  @Override
  public String toString() {
    return (String.format("ID(%d)", value));
  }

  private static final long serialVersionUID = 1L;

}
```



Step1 Dao (Java)

```java
@Dao(config = AppConfig.class)
public interface EmpDao {
  @Select
  Optional<Emp> selectById(ID<Emp> id);

  @Select
  List<Emp> selectAll();

  @Insert
  Result<Emp> insert(Emp emp);

  @Update
  Result<Emp> update(Emp emp);

  @Script
  void create();

}
```




Step1 Daoの利用 (Scala)

```scala
object SampleApp1 extends App {
  lazy val dao: EmpDao = new EmpDaoImpl()
  AppConfig.singleton.getTransactionManager.required({ () =>
    // create table & insert
    dao.create()
    val inserted = Seq(
      new Emp(ID.of(-1), "scott", 10, -1),
      new Emp(ID.of(-1), "allen", 20, -1)
    ).map(dao.insert)
    println(inserted)

    // idが2のEmpのageを +1 にupdate
    val updated =
      dao
        .selectById(ID.of(2))
        .map[Emp](_.grawOld) // Optional#mapは型推論が効かないため明示
        .map[Result[Emp]](dao.update)
    println(updated)

    dao.selectAll().forEach(println)
    // =>
    //   Emp(id=ID(1), name=scott, age=10, version=1)
    //   Emp(id=ID(2), name=allen, age=21, version=2)
  }: Runnable)
}
```



ビルドがそのままではうまくいかない

```bash
sbt:scala-doma-sample1> compile
[error] C:\scala-doma-sample1\src\main\scala\sample\SampleApp1.scala:7:30:
 not found: type EmpDaoImpl
[error]   lazy val dao: EmpDao = new EmpDaoImpl()
[error]                              ^
[error] one error found
[error] (compile:compileIncremental) Compilation failed
```

Javaから先にコンパイルしないといけない

⇒ build.sbtに追記
```scala
compileOrder := CompileOrder.JavaThenScala
```



指定して再コンパイル

```bash
sbt:scala-doma-sample1> compile
[error] C:\scala-doma-sample1\src\main\java\sample\EmpDao.java:19:1:
 エラー: [DOMA4019] ファイル[META-INF/sample/EmpDao/selectById.sql]が
 クラスパスから見つかりませんでした。ファイルの絶対パスは
 "C:\scala-doma-sample1\target\scala-2.12\classes\META-INF\sample\EmpDao\selectById.sql"です。
[error]   Optional<Emp> selectById(ID<Emp> id);
[error]                 ^^
```

コンパイルより前にsqlファイル（resources）を出力ディレクトリにコピーしないといけない

⇒ build.sbtに追記

```scala
compile in Compile := ((compile in Compile) dependsOn (copyResources in Compile)).value
```



```bash
sbt:scala-doma-sample1> compile
[info] Compiling 1 Scala source and 4 Java sources to C:\scala-doma-sample1\target\scala-2.12\classes ...
[info] Done compiling.
[success] Total time: 5 s, completed 2017/12/06 9:54:14
```
OK
```bash
sbt:scala-doma-sample1> run
...
select * from emp
12 06, 2017 9:55:26 午前 sample.EmpDaoImpl selectAll
情報: [DOMA2221] EXIT   : クラス=[sample.EmpDaoImpl], メソッド=[selectAll]
Emp(id=ID(1), name=scott, age=10, version=1)
Emp(id=ID(2), name=allen, age=21, version=2)
12 06, 2017 9:55:26 午前 org.seasar.doma.jdbc.tx.LocalTransaction commit
情報: [DOMA2067] ローカルトランザクション[1724208677]をコミットしました。
12 06, 2017 9:55:26 午前 org.seasar.doma.jdbc.tx.LocalTransaction commit
情報: [DOMA2064] ローカルトランザクション[1724208677]を終了しました。
[success] Total time: 2 s completed 2017/12/06 9:55:27
```
OK



Step1 build.sbt（完成版）

```diff
lazy val root = (project in file(".")).
  settings(
    inThisBuild(List(
      scalaVersion := "2.12.4",
      version      := "0.1.0-SNAPSHOT"
    )),
    name := "scala-doma-sample1",
    libraryDependencies ++= Seq(
      "org.seasar.doma" % "doma" % "2.19.0",
      "com.h2database" % "h2" % "1.4.193",
      scalaTest % Test
    )
  )

+ // for Doma annotation processor
+ compileOrder := CompileOrder.JavaThenScala
+ compile in Compile := ((compile in Compile) dependsOn (copyResources in Compile)).value
```



Step1の課題
 - アプリを作るのに2言語必要（Java, Scala）<!-- .element: class="fragment" -->
 - 特にコレクションが混ざるのがつらい<!-- .element: class="fragment" -->
 



Step2
-----
Doma関連クラスもScalaで作る



Step2 Entity (Scala)

```scala
// Doma2ではフィールドにアノテーションを付ける必要があるが
// コンストラクタパラメータに対してアノテーションを付けると
// パラメータに対してのみ有効になる
// 下記のように@fieldと明示することで自動的に作られる同名の
// 状態フィールドに対し アノテーションを有効にできる。
// http://www.scala-lang.org/api/current/scala/annotation/meta/index.html
@Entity(immutable = true)
case class Emp(
  @(Id @field)
  @(GeneratedValue @field)(strategy = GenerationType.SEQUENCE)
  @(SequenceGenerator @field)(sequence = "emp_id_seq")
  id: ID[Emp],
  name: String,
  age: Int,
  @(Version @field)
  version: Int
) {
  def growOld: Emp = this.copy(age = this.age + 1)
}

```



Step2 Domain  (Scala)

```scala
// Domainクラスにはvalueのgetter（getValue）が必要
// @scala.beans.BeanPropertyはgetterとsetterを作ってくれる
@Domain(valueType = classOf[Long])
case class ID[ENTITY](
  @BeanProperty value: Long)
```



case class
-----
EntityとDomainですが、アノテーションが少し増えてしまいましたが実装はだいぶ減りました

Scalaではcase classと定義するとhashCode()、equals()、toString()、等がコンパイル時に自動生成されるためこの恩恵を受けています



Step2 Dao (Scala)

```scala
// Scalaで作ったConfigはstaticメソッドを持てず
// アノテーション引数に指定できないため
// 利用者がコンストラクタパラメータで渡す
@Dao
trait EmpDao {
  @Select
  def selectById(id: ID[Emp]): Optional[Emp]

  @Select
  def selectAll: java.util.List[Emp]

  @Insert
  def insert(emp: Emp): Result[Emp]

  @Update
  def update(emp: Emp): Result[Emp]

  @Script
  def create(): Unit
}
```



Step2 Config (Scala)

```scala
object AppConfig extends Config {
  val dataSource = new LocalTransactionDataSource(
    "jdbc:h2:mem:tutorial;DB_CLOSE_DELAY=-1",
    "sa",
    null)
  val transactionManager = new LocalTransactionManager(
    dataSource.getLocalTransaction(getJdbcLogger))

  Class.forName("org.h2.Driver")

  override def getDialect: Dialect = new H2Dialect()

  override def getDataSource: DataSource = dataSource

  override def getTransactionManager: TransactionManager = transactionManager
}
```




Step2 Daoの利用 (Scala)

```scala
object SampleApp2 extends App {
  private lazy val dao: EmpDao = new EmpDaoImpl(AppConfig)
  AppConfig.getTransactionManager.required({ () =>
    // create table & insert
    dao.create()
    val inserted = Seq(
      Emp(ID[Emp](-1), "scott", 10, -1),
      Emp(ID[Emp](-1), "allen", 20, -1)
    ).map(dao.insert)
    println(inserted)

    // idが2のEmpのageを +1 にupdate
    val updated =
      dao
        .selectById(ID(2))
        .map[Emp](_.grawOld) // Optional#mapは型推論が効かないため明示
        .map[Result[Emp]](dao.update)
    println(updated)

    dao.selectAll.forEach(println)
    // =>
    //   Emp(ID(1),scott,10,1)
    //   Emp(ID(2),allen,21,2)
  }: Runnable)

}

```



ビルドがつらかった



Annotation Processingはjavac時に行われるが、

Doma関連クラスがScalaクラスになったため

scalacがコンパイラになり

javacがそもそも走らない




Doma2のAnnotation Processingの流れ（再掲）
![javac](./doma-javac.png)

<span style="font-size:60%">http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html </span>

 




どうしたか




ビルドを3フェーズに分けた

1. Doma関連クラスのソース(.scala)をコンパイル<!-- .element class="fragment" -->

1. Doma関連クラスの.classファイルを-proc:onlyでjavacしてAnnotation Processingのみ行う<!-- .element class="fragment" -->

1. 2.で自動生成されたソース(.java)と自動生成クラスの利用クラス(.scala)をコンパイル<!-- .element class="fragment" -->



Step2のビルドの流れ
![scalac](./doma-scalac.png)



Step2 project構成
```
project-root/
 - generate/         #自動生成先
 -repository
   - src/
      - main/
        - scala/     #Doma関連クラスのソース
        - resources/ #SQLファイル
   - target/
      - scala-2.12/
         - classes/  #Doma関連クラスのビルド結果の出力ディレクトリ
 - src/
    - main/
       - scala/      #Dao利用クラスのソース
 - target/
    - scala-2.12/
       - classes/    #Dao利用クラスのビルド結果の出力ディレクトリ
 - build.sbt
```



Step2 build.sbt

```scala
lazy val root = (project in file(".")).settings(
  inThisBuild(List(
    scalaVersion := "2.12.4",
    version      := "0.1.0-SNAPSHOT"
  )),
  name := "scala-doma-sample2",
  libraryDependencies += scalaTest % Test,
  // for Doma annotation processor
  (unmanagedSourceDirectories in Compile) += generateSourceDirectory
) dependsOn repository aggregate repository

lazy val repository = project.settings(
  inThisBuild(List(
    scalaVersion := "2.12.4",
    version      := "0.1.0-SNAPSHOT"
  )),
  name := "scala-doma-sample2-repository",
  libraryDependencies ++= Seq(
    "org.seasar.doma" % "doma" % "2.19.0",
    "com.h2database" % "h2" % "1.4.193",
    scalaTest % Test
  ),
  // for Doma annotation processor
  processAnnotationsSettings
)

lazy val processAnnotations = taskKey[Unit]("Process annotations")
lazy val generateSourceDirectory = file(".").getAbsoluteFile / "generated"

lazy val processAnnotationsSettings: Seq[Def.Setting[_]] = Seq(
  processAnnotations in Compile := {
    val classes = (unmanagedSources in Compile).value.filter(_.getPath.endsWith("scala"))
    val log = streams.value.log
    val separator = System.getProperties.getProperty("path.separator")
    val classpath = ((dependencyClasspath in Compile).value.files mkString separator) + separator + (classDirectory in Compile).value.toString
    if (classes.nonEmpty) {
      log.info("Processing annotations ...")
      deleteFiles(generateSourceDirectory)
      val cutSize = (scalaSource in Compile).value.getPath.length + 1
      val classesToProcess = classes.map(_.getPath.substring(cutSize).replaceFirst("\\.scala", "").replaceAll("[\\\\/]", ".")).mkString(" ")
      val directory = (classDirectory in Compile).value
      val command = s"javac -cp $classpath -proc:only -XprintRounds -d $directory -s $generateSourceDirectory $classesToProcess"
      executeCommand(command, "Failed to process annotations.", log)
      log.info("Done processing annotations.")
    }
  },
  processAnnotations in Compile := ((processAnnotations in Compile) dependsOn (compile in Compile)).value,
  copyResources in Compile := Def.taskDyn {
    val copy = (copyResources in Compile).value
    Def.task {
      (processAnnotations in Compile).value
      copy
    }
  }.value
)

def deleteFiles(targetDirectory: File): Unit = {
  def delete(f: File): Unit =
    if (f.isFile)
      f.delete()
    else
      f.listFiles.toList.foreach(delete)
  if(targetDirectory.exists)
    delete(targetDirectory)
  else
    targetDirectory.mkdir()
}

def executeCommand(command: String, errorMessage: => String, log: Logger): Unit = {
  val process = java.lang.Runtime.getRuntime.exec(command)
  printInputStream(process.getErrorStream, log)
  if (process.waitFor() != 0) {
    log.error(errorMessage)
    sys.error("Failed running command: " + command)
  }
}

def printInputStream(is: scala.tools.nsc.interpreter.InputStream, log: Logger): Unit = {
  val br = new java.io.BufferedReader(new java.io.InputStreamReader(is))
  try {
    var line = br.readLine
    while (line != null) {
      log.info(line)
      line = br.readLine
    }
  } finally {
    br.close()
  }
}
```



```bash
sbt:scala-doma-sample2> compile
[info] Compiling 4 Scala sources to C:\scala-doma-sample2\repository\target\scala-2.12\classes ...
[info] Done compiling.
[info] 往復1:
[info]  入力ファイル: {sample.AppConfig, sample.Emp, sample.EmpDao, sample.ID}
[info]  注釈: [scala.reflect.ScalaSignature, org.seasar.doma.Entity, org.seasar.doma.Id, org.seasar.doma.GeneratedValue, org.seasar.doma.SequenceGenerator, org.seasar.doma.Version, org.seasar.doma.Dao, org.seasar.doma.Script, org.seasar.doma.Select, org.seasar.doma.Insert, org.seasar.doma.Update, org.seasar.doma.Domain]
[info]  最後の往復: false
[info] 往復2:
[info]  入力ファイル: {sample._ID, sample._Emp, sample.EmpDaoImpl}
[info]  注釈: [javax.annotation.Generated, java.lang.SuppressWarnings, java.lang.Override]
[info]  最後の往復: false
[info] 往復3:
[info]  入力ファイル: {}
[info]  注釈: []
[info]  最後の往復: true
[info] Done processing annotations.
[info] Compiling 1 Scala source and 3 Java sources to C:\scala-doma-sample2\target\scala-2.12\classes ...
[info] Done compiling.
[success] Total time: 7 s, completed 2017/12/06 18:17:45
```
OK



```sh
sbt:scala-doma-sample2> run
...
情報: [DOMA2221] EXIT   : クラス=[sample.EmpDaoImpl], メソッド=[selectAll]
Emp(ID(1),scott,10,1)
Emp(ID(2),allen,21,2)
12 06, 2017 6:22:19 午後 org.seasar.doma.jdbc.tx.LocalTransaction commit
情報: [DOMA2067] ローカルトランザクション[1673739243]をコミットしました。
12 06, 2017 6:22:19 午後 org.seasar.doma.jdbc.tx.LocalTransaction commit
情報: [DOMA2064] ローカルトランザクション[1673739243]を終了しました。
[success] Total time: 5 s, completed 2017/12/06 18:22:20
```
OK



Step1の課題
 - アプリを作るのに2言語必要（Java, Scala）<!-- .element: class="fragment init-visible current-visible" data-fragment-index="1"--><span style="font-size:80%">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="font-size:150%">&nbsp;</span><!-- .element: class="fragment init-visible current-visible" data-fragment-index="1"-->
 - <!-- .element: class="fragment init-hedden" data-fragment-index="1"-->~~アプリを作るのに2言語必要（Java, Scala）~~ <!-- .element: class="fragment init-hedden data-fragment-index="1"" --><span style="color:green;font-size:150%">✔</span><!-- .element: class="fragment init-hedden data-fragment-index="1"" -->
 - 特にコレクションが混ざるのがつらい <p class="fragment">… 変わってない</p>

Step2の課題<!-- .element: class="fragment" -->
 - @field, @BeanPropertyは見にくくなるため使いたくない<!-- .element: class="fragment" -->
 - IDEプラグインが使えない(SQLファイルに飛べない)<!-- .element: class="fragment" -->



Step3
-----
Domala



Domala
-----

いままでの課題をクリアすべく、**Scala**による**Doma2**のWrapperを作りました。

**Domala**といいます。

まだベータ版ですが年度内には1.0リリースできるように目指してます。




Domalaの依存ライブラリ
-----

|  |  |
|:-----------|:------|
|Scala<!-- .element: style="font-size:70%" --> | 2.12.4<!-- .element: style="font-size:70%" -->|
|Doma2<!-- .element: style="font-size:70%" --> | "org.seasar.doma" % "doma" % "2.19.0"<!-- .element: style="font-size:70%" --> |
|scalameta<!-- .element: style="font-size:70%" --> | "org.scalameta" %% "scalameta" % "1.8.0"<!-- .element: style="font-size:70%" -->|
|scalameta-paradise<!-- .element: style="font-size:70%" --> | "org.scalameta" % "paradise" % "3.0.0-M10"<!-- .element: style="font-size:70%" -->|
|scala-reflect<!-- .element: style="font-size:70%" --> |  "org.scala-lang" % "scala-reflect" % "2.12.4"<!-- .element: style="font-size:70%" -->|
|  |  |




scalameta 
-----
- Scalaの準標準の実験的マクロライブラリ<!-- .element class="fragment" -->

- ASTの構築ができる<!-- .element class="fragment" -->

- scalameta paradiseと組み合わせてアノテーションマクロが組める<!-- .element class="fragment" -->



scalameta paradise
-----
- アノテーションマクロを実現するScalaのコンパイラプラグイン<!-- .element class="fragment" data-fragment-index="1" style="font-size:90%"-->

- JavaのPluggable Annotation Processing APIに近く、ASTの検証、及び改変ができる<!-- .element class="fragment" data-fragment-index="2" style="font-size:90%"-->

- 次はscalamacrosとして生まれ変わるかもしれない<!-- .element class="fragment" data-fragment-index="3" style="font-size:90%"-->

http://scala-lang.org/blog/2017/10/09/scalamacros.html<!-- .element class="fragment" data-fragment-index="3" style="font-size:90%"-->



scala-reflect
-----
- こちらもScalaの準標準の実験的マクロライブラリ<!-- .element class="fragment" style="font-size:80%"-->

- 歴史的に言うとscala-reflectがマクロv1, scalametaがマクロv2<!-- .element class="fragment" style="font-size:80%"-->

- scalameta paradiseのアノテーションマクロはアノテーションがつけられた対象のASTの情報のみしか現状取得できない。（型の詳細情報が取得できない）<!-- .element class="fragment" style="font-size:80%"-->

- このscala-reflectはコンパイル時のクラスローダーミラーが参照できるため大体なんでもできる（型の判定、定義メソッドの取得、ASTの構築、など）<!-- .element class="fragment" style="font-size:80%"-->

- ただし難解<!-- .element class="fragment" style="font-size:80%"-->



Domalaでのマクロの利用用途
-----
- scalameta / scalameta paradise
  - アノテーション対象の構文検証<!-- .element class="fragment" style="font-size:80%"-->
  - Dao実装、Entityヘルパーなどの自動生成<!-- .element class="fragment" style="font-size:80%"-->

- scala-reflect
  - scalametaでは判定できない型の検証<!-- .element class="fragment" style="font-size:80%"-->
  - 例えばIN句に渡されるパラメータはIterableのサブタイプであり、かつその中身はSQLにマッピングできる型であるか、のような検証<!-- .element class="fragment" style="font-size:80%"-->



Domalaのアノテーション
-----

| Doma2 |  | Domala |
|:-----------|:------:|:------------|
| Entity | ⇒ | domala.Entity |
| Domain | ⇒ | domala.Holder<span style="font-size:50%">*1</span> |
| Dao    | ⇒ | domala.Dao |

その他のorg.seasar.domaパッケージ内のアノテーションはdomala.XXとして同様に利用できます

(0.1.0-beta.6ではストアド系は未実装）

<span style="font-size:50%">*1</span><span style="font-size:70%">Doma3からDomainはHolderに名前が変わるためDomalaも倣いました</span>



Domalaの実装例



Domala project構成

```
project-root/
 - src/
    - main/
       - scala/     #Doma関連クラスのソース, Dao利用クラスのソース
 - target/
    - scala-2.12/
       - classes/   #ビルド結果の出力ディレクトリ
 - build.sbt
```



Domala build.sbt
```scala
lazy val root = (project in file(".")).settings(
  inThisBuild(List(
    scalaVersion := "2.12.4",
    version      := "0.1.0-SNAPSHOT"
  )),
  name := "scala-doma-sample3",
  libraryDependencies ++= Seq(
    "com.github.domala" %% "domala" % "0.1.0-beta.6",
    "com.h2database" % "h2" % "1.4.193",
    scalaTest % Test
  ),
  // for Domala annotation macro
  metaMacroSettings,
)
lazy val metaMacroSettings: Seq[Def.Setting[_]] = Seq(
  addCompilerPlugin("org.scalameta" % "paradise" % "3.0.0-M10" cross CrossVersion.full),
  scalacOptions += "-Xplugin-require:macroparadise",
  scalacOptions in (Compile, console) ~= (_ filterNot (_ contains "paradise")) // macroparadise plugin doesn't work in repl yet.
)
```



Domala Entity (Scala)

```scala
@Entity
case class Emp(
  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE)
  @SequenceGenerator(sequence = "emp_id_seq")
  id: ID[Emp],
  name: String,
  age: Int,
  @Version
  version: Int) {
  def growOld: Emp = this.copy(age = this.age + 1)
}
```



Domala Holder (Scala)

```scala
@Holder
case class ID[ENTITY](value: Long)
```



Domala Dao (Scala)

```scala
@Dao
trait EmpDao {
  @Select("select * from emp where id = /* id */0")
  def selectById(id: ID[Emp]): Option[Emp] // scala.Optionが使える

  @Select("select * from emp")
  def selectAll: Seq[Emp] // scala.Seqが使える

  @Insert
  def insert(emp: Emp): Result[Emp]

  @Update
  def update(emp: Emp): Result[Emp]

  @Script("""
  create table emp(
    id int not null primary key,
    name varchar(20),
    age int,
    version int not null
  );
  create sequence emp_id_seq start with 1;
  """)
  def create(): Unit

}
```



Scalaはヒアドキュメント記述ができるため、DomalaではSQLファイルは使わずアノテーションパラメータのリテラル文字列でSQLを記述します。
```scala
  @Select("""
  select
    e.id,
    e.name,
    e.dept_id,
    d.name as dept_name
  from
    emp e
    inner join
    dept d
    on (e.dept_id = d.id)
  where
    e.id = /*id*/0
  """)
  def selectWithDeptById(id: ID[Emp]): Option[EmpDept]
```



Domala Daoの利用 (Scala)

```scala
object SampleApp3 extends App {
  private implicit lazy val config: Config = AppConfig
  private lazy val dao: EmpDao = EmpDao.impl
  Required {
    // create table & insert
    dao.create()
    val inserted = Seq(
      Emp(ID(-1), "scott", 10, -1),
      Emp(ID(-1), "allen", 20, -1)
    ).map(dao.insert)
    println(inserted)

    // idが2のEmpのageを +1 にupdate
    val updated =
      dao
        .selectById(ID(2))
        .map(_.growOld) // Optionなので型推論が働く
        .map(dao.update)
    println(updated)

    dao.selectAll.foreach(println)
    // =>
    //   Emp(ID(1),scott,10,1)
    //   Emp(ID(2),allen,21,2)
  }

}
```



```bash
sbt:scala-doma-sample3> compile
[info] Updating {file:/C:/Users/19990013/Documents/@E_Scala/scala-doma-sample3/}root...
[info] Done updating.
[info] Compiling 5 Scala sources to C:\Users\19990013\Documents\@E_Scala\scala-doma-sample3\target\scala-2.12\classes ...
[info] Done compiling.
[success] Total time: 6 s, completed 2017/12/07 21:32:55
```

```bash
sbt:scala-doma-sample3> run
...
select * from emp
12 07, 2017 9:34:16 午後 sample.EmpDao selectAll
情報: [DOMA2221] EXIT   : クラス=[sample.EmpDao], メソッド=[selectAll]
Emp(ID(1),scott,10,1)
Emp(ID(2),allen,21,2)
12 07, 2017 9:34:16 午後 org.seasar.doma.jdbc.tx.LocalTransaction commit
情報: [DOMA2067] ローカルトランザクション[1853944763]をコミットしました。
12 07, 2017 9:34:16 午後 org.seasar.doma.jdbc.tx.LocalTransaction commit
情報: [DOMA2064] ローカルトランザクション[1853944763]を終了しました。
[success] Total time: 2 s, completed 2017/12/07 21:34:17
```



Step1の課題
 - ~~アプリを作るのに2言語必要（Java, Scala）~~ <span style="color:green;font-size:150%">✔</span>
 - 特にコレクションが混ざるのがつらい<!-- .element: class="fragment init-visible current-visible" data-fragment-index="1"--><span style="font-size:80%">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="font-size:150%">&nbsp;</span><!-- .element: class="fragment init-visible current-visible" data-fragment-index="1"-->
 - <!-- .element: class="fragment init-hedden" data-fragment-index="1" -->~~特にコレクションが混ざるのがつらい~~<!-- .element: class="fragment init-hedden" data-fragment-index="1" --><span style="color:green;font-size:150%">✔</span><!-- .element: class="fragment init-hedden" data-fragment-index="1" -->

Step2の課題
 - @field, @BeanPropertyは見にくくなるため使いたくない<!-- .element: class="fragment init-visible  current-visible" data-fragment-index="2"--><span style="font-size:80%">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="font-size:150%">&nbsp;</span><!-- .element: class="fragment init-visible current-visible" data-fragment-index="2"-->
 - <!-- .element: class="fragment init-hedden" data-fragment-index="2"-->~~@field, @BeanPropertyは見にくくなるため使いたくない~~<!-- .element: class="fragment init-hedden" data-fragment-index="2"--><span style="color:green;font-size:150%">✔</span><!-- .element: class="fragment init-hedden" data-fragment-index="2" -->
 - IDEプラグインが使えない(SQLファイルに飛べない)<!-- .element: class="fragment init-visible  current-visible" data-fragment-index="3"--><span style="font-size:80%">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="font-size:150%">&nbsp;</span><!-- .element: class="fragment init-visible current-visible" data-fragment-index="3"-->
 - <!-- .element: class="fragment init-hedden" data-fragment-index="3"-->~~IDEプラグインが使えない(SQLファイルに飛べない)~~<!-- .element: class="fragment init-hedden" data-fragment-index="3"--><span style="color:green;font-size:150%">✔</span><!-- .element: class="fragment init-hedden" data-fragment-index="3" -->



やっと全てScalaでDomaが使える環境ができました！



DomalaとIDE
-----

IntelliJ IDEA + Scala pluginでも開発できます

![idea-error](./idea-error.png)



アノテーション横の謎のボタン
![idea-expand1](./idea-expand1.png)



押すとマクロの結果が見れます

そのままデバッグもできます

![idea-expand2](./idea-expand2.png)



DomalaでStream検索
-----

```scala
  @Select(
    "select * from emp",
    strategy = SelectType.STREAM
  )
  def selectAll[R](mapper: Stream[Emp] => R): R
```

ScalaではFunction&lt;T, R&gt;は T => Rと書けます



DomalaでStream検索
-----

ScalaのStreamは気を付けて書かないとメモリを逼迫するのでSelectType.ITERATORを追加してます

toStreamでStreamに変換もできるのでこちらだけでもいいかもしれません

```scala
  @Select(
    "select * from emp where id in /* id */()",
    strategy = SelectType.ITERATOR
  )
  def selectByIds[R](id: Seq[ID[Emp]])(mapper: Iterator[Emp] => R): R

  def selectById(id: ID[Emp]): Option[Emp] = selectByIds(Seq(id)) {
    _.toStream.headOption
  }
```








✔
