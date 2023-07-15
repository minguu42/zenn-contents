---
title: "Goのクエリビルダーgoqu入門"
emoji: "✨"
type: "tech" # tech: 技術記事, idea: アイデア
topics: ["go", "goqu"]
published: true
---

## はじめに

この記事ではGoのクエリビルダーであるgoquの基本的な使い方を紹介したいと思います。

この記事が他の人の参考になったら幸いです。
また、この記事の内容に間違った記載がありましたら、指摘してもらえるとありがたいです。

## goquとは

[goqu](https://github.com/doug-martin/goqu)は表現力豊かなSQLビルダーです。また、SQLを実行することもできます。複数のSQL方言に対応しており、複数のレコードをスキャンして構造体やプリミティブ値にマッピングする機能もあります。

しかし、goquはORMとして使用されることを意図したものではないので、アソシエーションやフックの機能はありません。そのような機能が欲しい場合は[GORM](https://github.com/jinzhu/gorm)や[Hood](https://github.com/eaigner/hood)を使用することが推奨されています。

このようにgoquは単純な機能に集中したSQLビルダーです。そのため私のようにSQLを文字列として直接書くのは面倒臭いが、GORMなどは機能が複雑で難しいと感じた人にピッタリなSQLビルダーだと思います。

ここからはgoquを使って基本的なCRUDのSQLを生成する方法について紹介します。記載するコードは`Dialect`を使用してMySQL用のSQLを生成します。直接構造体をスキャンしたり、SQLを実行したい場合は公式ドキュメントの[Database](http://doug-martin.github.io/goqu/docs/database.html)を参照して下さい。

## 共通DSL

goquはそれぞれ`InsertDataset`、`SelectDataset`など各データセット構造体から`.ToSQL`メソッドでSQLを生成するところはどんなSQLを生成する場合でも共通です。
プリペアドステートメントを利用したい場合は`.Prepare`メソッドを利用します。

```go
func main() {
	dialect := goqu.Dialect("mysql")
	sql1, args1, err := dialect.From("users").Select("first_name").Where(goqu.C("id").Eq(1)).ToSQL()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(sql1, args1)
	// SELECT `first_name` FROM `users` WHERE (`id` = 1) []

	sql2, args2, err := dialect.From("users").Select("first_name").Where(goqu.C("id").Eq(1)).Prepared(true).ToSQL()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(sql2, args2)
	// SELECT `first_name` FROM `users` WHERE (`id` = ?) [1]
}
```

`ToSQL`メソッドの戻り値の`sql`は`string`型の文字列であり、`args`は`interface{}`型のスライスです。

## INSERT句を生成する

goquでINSERT句を生成する方法は複数あります。そのうちよく使用するものをここでは取り上げます。
他の方法は[Inserting](https://doug-martin.github.io/goqu/docs/inserting.html)を参照して下さい。

### `.Cols`、`.Vals`でINSERT句を生成する

`.Cols`、`.Vals`メソッドを使用してINSERT句を生成する方法です。
この方法はレコードの値を並べられるので、複数のレコードを挿入する際に便利です。

`.Insert`メソッドの引数にはテーブル名を渡します。

```go
func main() {
	dialect := goqu.Dialect("mysql")
	sql, _, err := dialect.Insert("users").Cols("first_name", "last_name", "age").Vals(
		goqu.Vals{"Yamada", "Taro", 20},
		goqu.Vals{"Yamada", "Hanako", 30},
	).ToSQL()
	fmt.Println(sql)
	// INSERT INTO `users` (`first_name`, `last_name`, `age`) VALUES ('Yamada', 'Taro', 20), ('Yamada', 'Hanako', 30)
}
```

### `goqu.Record`でINSERT句を生成する

`goqu.Record`を使用してINSERT句を生成する方法です。
この方法はカラムと対応する値を分かりやすく書けます。

```go
func main() {
	dialect := goqu.Dialect("mysql")
	sql, _, err := dialect.Insert("users").Rows(
		goqu.Record{"first_name": "Yamada", "last_name": "Taro", "age": 20},
		goqu.Record{"first_name": "Yamada", "last_name": "Hanako", "age": 30},
	).ToSQL()
	fmt.Println(sql)
	// INSERT INTO `users` (`age`, `first_name`, `last_name`) VALUES (20, 'Yamada', 'Taro'), (30, 'Yamada', 'Hanako')
}
```

### 構造体でINSERT句を生成する

構造体と`db`タグを使用してINSERT句を生成する方法です。
この方法はDBのテーブルに対応する構造体を使用する場合にとても便利です。

構造体はフィールドをエクスポートし、`db`タグで対応するテーブルのカラム名を指定します。
`db`タグの値を`-`にするとフィールドがエクスポートされていてもgoquから無視されます。
`goqu`タグはその他のオプションを利用する場合に使用します。
`skipinsert`は挿入時にそのフィールドを無視し、`defaultifempty`はそのフィールドの値がゼロ値であった場合に`DEFAULT`を使用します。

```go
type User struct {
	ID        int    `db:"id"  goqu:"skipinsert"`
	FirstName string `db:"first_name" goqu:"defaultifempty"`
	LastName  string `db:"last_name"`
	Age       int    `db:"age"`
	Address   string `db:"-"`
}

func main() {
	user1 := User{
		FirstName: "Yamada",
		LastName:  "Taro",
		Age:       20,
		Address:   "Japan",
	}
	dialect := goqu.Dialect("mysql")
	sql, _, err := dialect.Insert("users").Rows(
		user1,
		User{
			LastName: "Hanako",
			Age:      30,
			Address:  "Japan",
		},
	).ToSQL()
	fmt.Println(sql)
	// INSERT INTO `users` (`age`, `first_name`, `last_name`) VALUES (20, 'Yamada', 'Taro'), (30, DEFAULT, 'Hanako')
}
```

## SELECT句を生成する

goquでSELECT句を生成するには`.From`、`.Select`メソッドを使用します。
上記のメソッドに加えて`.Where`、`Limit`、`Offset`メソッドを使用することで目的に叶うSQLを生成します。
また、複雑なSELECT句もgoquは生成できますが、ここでは紹介しきれないので詳しくは[Selecting](https://doug-martin.github.io/goqu/docs/selecting.html)を参照して下さい。列を構造体にマッピングする`ScanStruct`なども先のドキュメントに記載されています。

基本的なSELECT句は以下のようになります。
`.From`メソッドの引数にはテーブル名を渡します。
構造体のポインタを渡すことで構造体のフィールドを全て指定できます。
ただし、`db`タグの値が`-`の場合は無視され、フィールドはアルファベット順に並ぶことに注意して下さい。

```go
type User struct {
	ID        int    `db:"id"  goqu:"skipinsert"`
	FirstName string `db:"first_name" goqu:"defaultifempty"`
	LastName  string `db:"last_name"`
	Age       int    `db:"age"`
	Address   string `db:"-"`
}

func main() {
	dialect := goqu.Dialect("mysql")
	sql1, _, err := dialect.From("users").Select("first_name", "last_name", "age").Where(goqu.C("age").Eq(20)).ToSQL()
	fmt.Println(sql1)
	// -> SELECT `first_name`, `last_name`, `age` FROM `users` WHERE (`age` = 20)

	sql2, _, err := dialect.From("users").Select(&User{}).Where(
		goqu.C("first_name").Eq("Taro"),
		goqu.C("age").Gt(20),
		goqu.C("created_at").IsNotNull(),
	).ToSQL()
	fmt.Println(sql2)
	// SELECT `age`, `first_name`, `id`, `last_name` FROM `users` WHERE ((`first_name` = 'Taro') AND (`age` > 20) AND (`created_at` IS NOT NULL))
}
```

また、`.Where`メソッドを重ねることもできます。

```go
func main() {
	dialect := goqu.Dialect("mysql")
	sql, _, err := dialect.From("users").Select("first_name", "last_name", "age").
		Where(goqu.C("age").Eq(20)).
		Where(goqu.C("age").IsNull()).
		ToSQL()
	fmt.Println(sql)
	// SELECT `first_name`, `last_name`, `age` FROM `users` WHERE ((`age` = 20) AND (`age` IS NULL))
}
```

## Update句を生成する

goquでUpdate句を生成するには`.Update`、`.Set`メソッドを使用します。
セットする値の指定には主に`goqu.Record`か構造体を使用します。

`.Update`メソッドの引数にテーブル名を渡します。
Insert句を生成する場合と同様に構造体のフィールドに`db`、`goqu`タグを使用できます。
`goqu`タグの`skipupdate`はUpdate句の生成時にそのフィールド値を無視します。

```go
type User struct {
	ID        int    `db:"id"  goqu:"skipinsert,skipupdate"`
	FirstName string `db:"first_name" goqu:"defaultifempty"`
	LastName  string `db:"last_name"`
	Age       int    `db:"age"`
	Address   string `db:"-"`
}

func main() {
	dialect := goqu.Dialect("mysql")
	sql1, _, err := dialect.Update("users").Set(
		goqu.Record{"first_name": "Jiro", "last_name": "Sato"},
	).ToSQL()
	fmt.Println(sql1)
	// UPDATE `users` SET `first_name`='Jiro',`last_name`='Sato'

	user := User{
		ID:       100,
		LastName: "Jiro",
		Age:      21,
	}
	sql2, _, err := dialect.Update("users").Set(user).Where(goqu.C("id").Eq(1)).ToSQL()
	fmt.Println(sql2)
	// UPDATE `users` SET `age`=21,`first_name`=DEFAULT,`last_name`='Jiro' WHERE (`id` = 1)
}
```

また、`goqu.Record`は`map[string]interface{}`型のエイリアスであるので以下のように扱えます。

```go
func main() {
	dialect := goqu.Dialect("mysql")
	record := goqu.Record{}
	record["id"] = 100
	record["first_name"] = "Jiro"
	sql, _, err := dialect.Update("users").Set(record).ToSQL()
	fmt.Println(sql)
	// UPDATE `users` SET `first_name`='Jiro',`id`=100
}
```

## DELETE句を生成する

goquでDELETE句を生成するには`.Delete`メソッドを使用します。
より詳しくは[Deleting](https://doug-martin.github.io/goqu/docs/deleting.html)を参照して下さい。

```go
func main() {
	dialect := goqu.Dialect("mysql")
	sql1, _, _ := dialect.Delete("users").ToSQL()
	fmt.Println(sql1)
	// DELETE `users` FROM `users`

	sql2, _, _ := dialect.Delete("users").Where(goqu.C("id").Eq(1)).ToSQL()
	fmt.Println(sql2)
	// DELETE `users` FROM `users` WHERE (`id` = 1)
}
```

## 参考

- [goqu](https://github.com/doug-martin/goqu)
- [Inserting](https://doug-martin.github.io/goqu/docs/inserting.html)
- [Selecting](https://doug-martin.github.io/goqu/docs/selecting.html)
- [Updating](https://doug-martin.github.io/goqu/docs/updating.html)
- [Deleting](https://doug-martin.github.io/goqu/docs/deleting.html)
