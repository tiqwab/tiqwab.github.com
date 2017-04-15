---
layout: post
title: 「Spring Boot ハンズオン」 と戯れる
tags: "spring boot, java"
comments: true
---

最近 Spring Boot のおさらいをかねて [Spring Boot ハンズオン][14] を追いかけているのですが、その中でいくつか「これなんだっけ」、「これどうやればいいんだっけ」といった点が出てきたので整理しました。

主に Spring MVC や JPA まわりの話になっています。

1. [ModelAttribute](#anchor1)
2. [RedirectAttributes](#anchor2)
3. [Bean Validation](#anchor3)
4. [JPA with JSR 310](#anchor4)

### 環境

- Spring Boot: 1.5.2.RELEASE

<a id="anchor1"></a>

### 1. ModelAttribute

Spring Boot ハンズオンでは サーバでのフォーム処理において `@ModelAttribute` を付加したメソッドを使用しています ([AccountController][1])。

`@ModelAttribute` の主な役割の 1 つはリクエストパラメータを自動的に Model へマッピングして扱いやすくすることです。

サンプルとして以下のような Controller を作成しました。

- paramGet: `@RequestParam` を使用し、GET リクエストの処理を行う
- paramPost: `@RequestParam` を使用し、 POST リクエストの処理を行う
- modelGet: `@ModelAttribute` を使用し、 GET リクエストの処理を行う
- modelGet: `@ModelAttribute` を使用し、 POST リクエストの処理を行う

いずれの場合も受け取ったパラメータを確認のためログに吐き出しています。

```java
@Slf4j
@Controller
@RequestMapping("sample")
public class ModelSampleController {

    @RequestMapping(value = "param", method = RequestMethod.GET)
    public String paramGet(@RequestParam("name") String name, @RequestParam("email") String email) {
        log.info("paramGet: name: {}, email: {}", name, email);
        return "sample/param";
    }

    @RequestMapping(value = "param", method = RequestMethod.POST)
    public String paramPost(@RequestParam("name") String name, @RequestParam("email") String email) {
        log.info("paramPost: name: {}, email: {}", name, email);
        return "sample/param";
    }

    // Object with @ModelAttribute is created for each request
    @RequestMapping(value = "model", method = RequestMethod.GET)
    public String modelGet(@ModelAttribute UserModel user) {
        log.info("modelGet: name: {}, email: {}", user.getName(), user.getEmail());
        return "sample/model";
    }

    @RequestMapping(value = "model", method = RequestMethod.POST)
    public String modelPost(@ModelAttribute UserModel user) {
        log.info("modelPost: name: {}, email: {}", user.getName(), user.getEmail());
        return "sample/model";
    }

}

@Getter
@Setter
@NoArgsConstructor
class UserModel {
    private String name;
    private String email;
}
```

`/sample/param` , `/sample/model/` に対しそれぞれ GET , POST リクエストを投げます。

```
# GET to /sample/param
> curl 'http://localhost:8080/sample/param?name=aaa&email=bbb' -u 'user:f25b89f2-7a6e-4f27-946e-0f7fda45a072'
# POST to /sample/param
> curl 'http://localhost:8080/sample/param' -u 'user:f25b89f2-7a6e-4f27-946e-0f7fda45a072' -d 'name=aaa' -d 'email=bbb'
# GET to /sample/model
> curl 'http://localhost:8080/sample/model?name=aaa&email=bbb' -u 'user:f25b89f2-7a6e-4f27-946e-0f7fda45a072'
# POST to /sample/model
> curl 'http://localhost:8080/sample/model' -u 'user:f25b89f2-7a6e-4f27-946e-0f7fda45a072' -d 'name=aaa' -d 'email=bbb'
```

アプリケーションの出すログを見るといずれの場合もちゃんとパラメータの取得ができていることが確認できます。

```
c.t.example.app.ModelSampleController    : paramGet: name: aaa, email: bbb
c.t.example.app.ModelSampleController    : paramPost: name: aaa, email: bbb
c.t.example.app.ModelSampleController    : modelGet: name: aaa, email: bbb
c.t.example.app.ModelSampleController    : modelPost: name: aaa, email: bbb
```

`@ModelAttribute` はメソッドにつけることもできます。
その場合 `@RequestMapping` が付与された Controller 内 のメソッドを実行する前にパラメータのマッピングを行うようです。

```java
    @ModelAttribute
    public UserModel createUserModel() {
        return new UserModel();
    }

    @RequestMapping(value = "model", method = RequestMethod.GET)
    public String modelGet(UserModel user) {
        log.info("modelGet: name: {}, email: {}", user.getName(), user.getEmail());
        return "sample/model";
    }
```

また `@ModelAttribute` を付加したオブジェクトは自動的に `Model` に追加されるため、 View のレンダリング時に使用することができます。

```html
    <div>
        <p th:text="${userModel.name}">名前</p>
        <p th:text="${userModel.email}">E-mail</p>
    </div>
```

<a id="anchor2"></a>

### 2. RedirectAttributes

Spirnt Boot ハンズオンの `AccountController` では フォーム画面からの POST リクエスト処理後にリダイレクトで画面遷移しています ([PRG pattern][9])。

このとき Controller メソッドの引数に `RedirectAttributes` というクラスが登場しています。
`RedirectAttributes` の役割はリダイレクト時の情報の受け渡しです。

```java
@Controller
@RequestMapping("account")
public class AccountController {

    /* ... */

    @RequestMapping(value = "create", method = RequestMethod.POST)
    String create(@Validated AccountForm form, BindingResult bindingResult, RedirectAttributes attributes) {
        if (bindingResult.hasErrors()) {
            return "account/createForm";
        }

        Account account = Account.builder()
                .name(form.getName())
                .password(Account.PASSWORD_ENCODER.encode(form.getPassword()))
                .email(form.getEmail())
                .birthDay(form.getBirthDay())
                .zip(form.getZip())
                .address(form.getAddress())
                .age(form.getAge())
                .build();
        accountService.register(account);

        attributes.addFlashAttribute(account);
        // post-request-get (PRG) pattern
        return "redirect:/account/create?finish"; 
    }

    /* ... */

}
```

`RedirectAttributes` を使用しない場合を考えてみます。

単純にフォーム入力を受付、リダイレクトする処理を実装すると以下のような感じになると思います。

- (1) フォーム画面の View を返す
- (2) フォームからの POST 処理。ここではリダイレクトするだけ。
- (3) リダイレクト先。 返す View の中で POST された内容を使いたい。

```java
@Slf4j
@Controller
@RequestMapping("sample")
public class ModelSampleController {

    @ModelAttribute
    public UserModel createUserModel() {
        return new UserModel();
    }

    // (1)
    @RequestMapping(value = "redirect", params = "form", method = RequestMethod.GET)
    public String userFormRedirect() {
        return "sample/redirectFrom";
    }

    // (2)
    @RequestMapping(value = "redirect", method = RequestMethod.POST)
    public String redirectUser(UserModel user) {
        log.info("redirectUser: name: {}, email: {}", user.getName(), user.getEmail());
        return "redirect:/sample/redirect?success";
    }

    // (3)
    @RequestMapping(value = "redirect", params = "success", method = RequestMethod.GET)
    public String redirected(UserModel user) {
        log.info("redirected: name: {}, email: {}", user.getName(), user.getEmail());
        return "sample/redirectTo";
    }

}
```

Spring MVC では View 名が `redirect:` ではじまる場合、指定先へとリダイレクトさせます。
このときはクライアントに 302 ステータスコードを返しリダイレクト処理を行わせるため、そのままでは (2) のメソッド内で使用している情報は (3) へ引き継ぐことができません (ちなみにこの場合のステータスコードは本来 302 (Found) ではなく 303 (See Other) の方が適当なはず)。

実際ログを見るとリダイレクト先では UserModel の情報が渡されていないことがわかります。

```
redirectUser: name: aaa, email: bbb
redirected: name: null, email: null
```

リダイレクト先で使用したい情報がある場合、リダイレクト元のメソッドで `RedirectAttributes#addFlashAttribute` を使用します。
flash スコープなためリダイレクト先で更新するとその情報は再度取得できません。

```java
    // メソッドの引数に `RedirectAttributes` を追加
    @RequestMapping(value = "redirect", method = RequestMethod.POST)
    public String redirectUser(UserModel userModel, RedirectAttributes redirectAttributes) {
        log.info("redirectUser: name: {}, email: {}", userModel.getName(), userModel.getEmail());

        // redirectAttributes` に userModel を追加
        redirectAttributes.addFlashAttribute(userModel);
        return "redirect:/sample/redirect?success";
    }
```

`RedirectAttributes` には `addAttribute` というメソッドもあり、こちらを使用しても同様な処理ができます。
この場合、リダイレクト先は `/sample/redirect?success&name=aaa&email=bbb` のようになり、 URL にパラメータが付加されます。
そのためリダイレクト先で更新しても、再度同じ情報が利用できます。

```java
    @RequestMapping(value = "redirect", method = RequestMethod.POST)
    public String redirectUser(UserModel userModel, RedirectAttributes redirectAttributes) {
        log.info("redirectUser: name: {}, email: {}", userModel.getName(), userModel.getEmail());

        // URL にパラメータを埋め込むので、明示的に key と value の文字列表記を指定したほうが良さそう
        // 一応 value 側は toString() した上で自動的に URL encoding されるようだけれど
        redirectAttributes.addAttribute("name", userModel.getName());
        redirectAttributes.addAttribute("email", userModel.getEmail());
        return "redirect:/sample/redirect?success";
    }
```

<a id="anchor3"></a>

### 3. Bean Validation

Spring Boot ハンズオンでは入力フォームのサーバ側での検証のために [Bean Validation][2] を行っています。

#### 基本

Spring Boot では `spring-boot-starter-web` が依存関係に入っていれば、あらかた必要なものは用意してくれるので、簡単に Bean Validation をかけることができます。

- (1) Bean Validation を設定したいクラスに Annotation を付加する
- (2) Controller の ModelAttribute として使用している引数に `@Validated` を付加する

```java
@Slf4j
@Controller
@RequestMapping("sample")
public class ModelSampleController {

    @ModelAttribute
    public UserModel createUserModel() {
        return new UserModel();
    }

    @RequestMapping(value = "validate", params = "form", method = RequestMethod.GET)
    public String userForm() {
        return "sample/validate";
    }

    // (2)
    @RequestMapping(value = "validate", method = RequestMethod.POST)
    public String validateUser(@Validated UserModel user, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return "sample/validate";
        }
        return "sample/success";
    }

}

@Getter
@Setter
@NoArgsConstructor
class UserModel {
    // (1)
    @NotNull @Size(min = 1, max = 40)
    private String name;
    // (1)
    @NotNull @Size(min = 1) @Email
    private String email;
}
```

Controller の Bean Validation を行っているメソッド内では、 `bindingResult.hasErrors()` で検証処理の結果を確認し、エラーがあればもとのフォーム入力のページに戻しています。

フォーム側では `th:if` と `th:errors` でエラー時の処理を加えます ([参考][4])。

- (1) `th:if` でタグを表示する条件を指定する。 `${...}` 内部はいわゆる [Spring Expression Language (SpEL)][3] 形式
- (2) `th:errors` で thymeleaf が該当する項目のエラーを表示してくれる

```html
    <form th:action="@{/sample/validate}" method="post">
        <div>
            <label for="name">Name:</label>
            <input id="name" type="text" th:field="${userModel.name}" name="name" />
            <!-- (1), (2) -->
            <span th:if="${#fields.hasErrors('userModel.name')}" th:errors="${userModel.name}">errors</span>
        </div>
        <div>
            <label for="email">E-mail:</label>
            <input id="email" type="text" th:field="${userModel.email}" name="email" />
            <!-- (1), (2) -->
            <span th:if="${#fields.hasErrors('userModel.email')}" th:errors="${userModel.email}">errors</span>
        </div>
        <button type="submit">Submit</button>
    </form>
```

`userModel` を何回も書くのが面倒だという場合は、以下の記法も OK なようです。

```html
    <!-- Add 'th:object' attribute -->
    <form th:action="@{/sample/validate}" th:object="${userModel}" method="post">
        <div>
            <label for="name">Name:</label>
            <input id="name" type="text" th:field="*{name}" name="name" />

            <!-- Remove 'userModel' and use '*{...}' format -->
            <span th:if="${#fields.hasErrors('name')}" th:errors="*{name}">errors</span>
        </div>
        ...
    </form>
```

#### エラーメッセージの設定

エラー時にフォーム上に表示されるメッセージがデフォルトではユーザに優しくないので必要に応じて変更します。

`classpath:messages.properties` に該当するメッセージを設定すれば OK です。
(ただしロケール処理をちゃんとやろうとすると、多少イジイジする箇所が増えるかもです ([参考][5]))

メッセージの設定の詳細は[こちら][6]が参考になります。

```properties
# <AnnotationName>.<FormAttributeName>.<PropertyName>=<Message>
Size=長さは {2} 以上、 {1} 以下にしてください
Email=E-mail の形式が不正です
Pattern.zip=7 桁の整数を入力してください
```

#### 日付のフォーマット

Bean へのマッピングに `LocalDate` 等の日付を表すクラスを指定している場合、フォーマットを指定しないと変換が行なえません。

- (1) `@DateTimeFormat` で日付フォーマットの指定を行う
  - `iso` による [ISO に基づいた表記][7]
  - `pattern` による [SimpleDateFormat 表記][8]

```java
@Getter
@Setter
@NoArgsConstructor
class UserModel {
    @NotNull @Size(min = 1, max = 40)
    private String name;
    @NotNull @Size(min = 1) @Email
    private String email;
    // (1)
    @NotNull @DateTimeFormat(iso = DateTimeFormat.ISO.DATE /* or pattern = "yyyy/MM/dd" */)
    private LocalDate birthDay;
}
```

#### 型の変換時のエラーメッセージ

上で日付の変換が行えるようにしましたが、そのままだとエラー時に `IllegalArgumentException` といった内容がベタに表示されてあまりよろしくないです。

型変換時のエラーメッセージはメッセージファイルに `typeMismatch.<Type>=<Message>` のような形式で設定できます。

```
typeMismatch.java.time.LocalDate=日付の形式が異なります
```

<a id="anchor4"></a>

### 4. JPA with JSR 310

Java SE 8 から [JSR 310][12] の仕様に基づいて Date and Time API が提供されています。
その中でよく使用する日付型に以下のようなものがあります。

- `java.time.LocalDate`
- `java.time.LocalTime`
- `java.time.LocalDateTime`

これらを JPA Entity の項目に使用した場合、そのままだと binary としてカラムへマッピングされるようです。

```java
@Entity
public class Account {

    @Id @GeneratedValue
    private Long Id;
    private String name;
    private LocalDate birthDay; // mapped to binary column

    public Account() {

    }

}
```

これを防ぐには [`javax.persistence.AttributeConverter`][13] を実装して Entity の field とカラムとの間の変換を定義する必要があります。
自身で AttributeConverter を作成することもできますが ([参考][11]) 、上記の項目に関してならば現在は Spring 側で用意されているものを使用するほうが better だと思います。

- (1) `@EntityScan` を足し、 `Jsr310JpaConverters.class` を指定する ([参考][10])

```java
import org.springframework.data.jpa.convert.threeten.Jsr310JpaConverters;

@SpringBootApplication
// (1)
@EntityScan(basePackageClasses = {DemoApplication.class, Jsr310JpaConverters.class})
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

[1]: https://github.com/making/jsug-shop/blob/ch02/src/main/java/jsug/app/account/AccountController.java "AccountController"
[2]: http://beanvalidation.org/ "Bean Validation"
[3]: https://docs.spring.io/spring/docs/current/spring-framework-reference/html/expressions.html "SpEL"
[4]: http://www.thymeleaf.org/doc/tutorials/2.1/thymeleafspring.html#validation-and-error-messages
[5]: http://area-b.com/blog/2015/01/31/2332/
[6]: http://terasolunaorg.github.io/guideline/5.3.0.RELEASE/ja/ArchitectureInDetail/WebApplicationDetail/Validation.html#validation-message-def
[7]: https://ja.wikipedia.org/wiki/ISO_8601
[8]: https://docs.oracle.com/javase/jp/8/docs/api/java/text/SimpleDateFormat.html
[9]: http://qiita.com/furi/items/a32c106e9d7c4418fc9d
[10]: http://qiita.com/tag1216/items/25a64bba2bde98ea88d3
[11]: http://blog.beaglesoft.net/entry/2016/11/16/163000
[12]: https://jcp.org/en/jsr/detail?id=310
[13]: https://docs.oracle.com/javaee/7/api/javax/persistence/AttributeConverter.html
[14]: http://jsug-spring-boot-handson.readthedocs.io/en/latest/index.html
