ScalaでDomaを使ってみた話
------
Doma勉 2017



自己紹介
------
- スクィーラ
- 福岡のSier勤務
- 受託業務アプリ開発 / パッケージ開発
- 主にJava, 最近は若干Scala
- Doma歴4年くらい （S2Dao含めると10年強）



なぜScalaからDomaを使おうとしたのか
-----
- 現在チームではPlay-Java + Doma2がメイン構成  ⇒ Scalaに移行したい

- 他メンバーにPlay, Scala, ＋他のなにかのOR-Mapperを全て覚えさせるのはハードルが高いが、使い慣れたDomaを使えるのであれば移行がスムーズになるのではと思った



Scala＋Domaの話をする前に、
Pluggable Annotation Processing APIについて話します



Pluggable Annotation Processing API
-----

- Java標準のアノテーションマクロ

- **コンパイル時** に
  - アノテーションが付与されたコードの検証
  - Javaソースコードの自動生成

  が行える



DomaのAnnotation Processing
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
  implements EmpDao { .../* 実装 */ }
```



なにが嬉しいか
-----
- コンパイル時にコード検証
  - ガイド読まない人でも作れる
  - 実行時にエラーが先送りされない

- 自動生成
  - 事前にリフレクションを済ませてあるため実行時リフレクションがいらない
  - ボイラープレートが減る



Pluggable Annotation Processing APIを踏まえた上でScalaの話



Step1
----- 
Doma関連クラスはJavaで作り、利用をScalaで

Step2
-----
Doma関連クラスもScalaで作る

Step3
-----
Domala



Step1
----- 
Doma関連クラスはJavaで作り、利用をScalaで




Stes1では下記のJavaアプリをScalaにします 

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
      ).map(
        dao::insert
      ).collect(Collectors.toList());
      System.out.println(inserted);
      // idが2のEmpのageを +1
      final Optional<Result<Emp>> updated =
        dao
          .selectById(ID.of(2))
          .map(emp ->
            dao.update(emp.grawOld())
          );
      System.out.println(updated);
      final List<Emp> list = dao.selectAll();
      list.forEach(System.out::println);
      // =>
      //   Emp(id=ID(1), name=scott, age=10, version=1)
      //   Emp(id=ID(2), name=allen, age=21, version=2)
    });
  }
}
```



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



Step1 build.sbt（初期版 修正しないと動かない）
```scala
lazy val root = (project in file(".")).
  settings(
    inThisBuild(List(
      scalaVersion := "2.12.4",
      version      := "0.1.0-SNAPSHOT"
    )),
    name := "scala-doma-sample1",
    libraryDependencies += scalaTest % Test,
    libraryDependencies ++= Seq(
      "org.seasar.doma" % "doma" % "2.19.0",
      "com.h2database" % "h2" % "1.4.193"
    )
  )
```



Step1 Entity (Java)

```java
@Entity(immutable = true)
public class Emp implements Serializable {
  private static final long serialVersionUID = 1L;

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
      return emp.id.equals(id) && emp.name.equals(name) 
        && emp.age == age && emp.version == version;
    } else {
      return false;
    }
  }

  @Override
  public String toString() {
    return (String.format("Emp(id=%s, name=%s, age=%d, version=%d)",
        id, name, age, version));
  }

}

```



Step1 Domain  (Java)

```java
@Domain(valueType = long.class, factoryMethod = "of")
public final class ID<ENTITY> implements Serializable {
  private static final long serialVersionUID = 1L;
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
}
```



Step1 Dao (Java)

```java
@Dao(config = AppConfig.class)
public interface EmpDao {
  @Script
  void create();

  @Select
  Optional<Emp> selectById(ID<Emp> id);

  @Select
  List<Emp> selectAll();

  @Insert
  Result<Emp> insert(Emp emp);

  @Update
  Result<Emp> update(Emp emp);

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
    ).map {
      dao.insert
    }
    println(inserted)
    // idが2のEmpのageを +1
    val updated: Optional[Result[Emp]] = // Optionalは型推論効かない
      dao                                
        .selectById(ID.of(2))
        .map { emp =>
          dao.update(emp.grawOld)
        }
    println(updated) 
    val list = dao.selectAll()
    list.forEach(println)
    // =>
    //   Emp(id=ID(1), name=scott, age=10, version=1)
    //   Emp(id=ID(2), name=allen, age=21, version=2)
  }: Runnable)
}
```



ビルドがそのままではうまくいかない

```sh
>compile
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

```sh
>compile
[error] C:\scala-doma-sample1\src\main\java\sample\EmpDao.java:19:1:
 エラー: [DOMA4019] ファイル[META-INF/sample/EmpDao/selectById.sql]が
 クラスパスから見つかりませんでした。ファイルの絶対パスは
 "C:\scala-doma-sample1\target\scala-2.12\classes\META-INF\sample\EmpDao\selectById.sql"です。
[error]   Optional<Emp> selectById(ID<Emp> id);
[error]                 ^^
```

コンパイルより前にsqlファイル（resources）を出力ディレクトリにコピーしないといけない

build.sbtに追記
```scala
compile in Compile := ((compile in Compile) dependsOn (copyResources in Compile)).value
```



```sh
> compile
[info] Compiling 1 Scala source and 4 Java sources to C:\scala-doma-sample1\target\scala-2.12\classes ...
[info] Done compiling.
[success] Total time: 5 s, completed 2017/12/06 9:54:14
```
OK
```sh
> run
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
    libraryDependencies += scalaTest % Test,
    libraryDependencies ++= Seq(
      "org.seasar.doma" % "doma" % "2.19.0",
      "com.h2database" % "h2" % "1.4.193"
    )
  )

+ // for Doma annotation processor
+ compileOrder := CompileOrder.JavaThenScala
+ compile in Compile := ((compile in Compile) dependsOn (copyResources in Compile)).value
```



Step1の課題
 - アプリを作るのに2言語必要（Java, Scala） 
 - 特にコレクションが混ざるのがつらい
 






✔
