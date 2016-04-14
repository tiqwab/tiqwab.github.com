---
layout: post
title:  "JavaのテストにおけるMock使用例 with Spring and Camel"
tags: "java, test, mock, stub, spring-framework, apache-camel"
---

業務ではJavaを使用した開発が多いが、単体テストを書いていてMockやStubといったものを使用したことがないことに気がついたため、その使用例をまとめてみる。  

- [Mock, Stubとは](#anchor1)
- [JUnitでMockitoを使用したテスト](#anchor2)
- [Spring FrameworkとMockを使用したテスト](#anchor3)
- [Apache CamelとMockを使用したテスト](#anchor4)

---

<a id="anchor1"></a>

## Mock, Stubとは

厳密な定義はわからないが、[こちら][1]の記事がとても参考に。相互作用に着目するテストで使いたくなるのがMock、状態に着目するテストで欲しくなるのがStubということみたい。  

> この記事のタイトル "Mocks Aren't Stubs"にもかかわらず、モックとスタブの違いは本当は一番の問題ではない。最も興味深いのは、相互作用スタイル対状態スタイルというところだ。相互作用中心のテストを行うテスターは全てのサブオブジェクトについてモックを作る。状態中心のテスターは、実際のオブジェクトを使うのが現実的でないものについてのみスタブを作る。例えば、外部サービスやコストのかかるもの、状態中心のやり方では扱いにくいキャッシュのようなものだ。  

<a id="anchor2"></a>

## JUnitでMockitoを使用したテスト

JUnitをMock、Stub作成のためのライブラリであるMockitoと組み合わせて使用してみた。参考にしたのは[こちら][2]と[こちら][3]。使用例は上の記事に合わせ、倉庫からの商品の引当ロジック(注文に対して在庫が充分に存在するかを判断)のテストとした。

### 状態中心のテスト(Stub使用)

{% highlight java %}
public class OrderStateTest {

    private static final String TALISKER = "tarisker";

    @Test
    public void testOrderIsFilledIfEnoughInWarehouse() throws Exception {
        // setup
        Warehouse warehouse = mock(Warehouse.class);
        when(warehouse.getInventory(TALISKER)).thenReturn(50);
        Order order = new Order(TALISKER, 20);
        // execute
        order.fill(warehouse);
        // verify
        assertThat(order.isFilled(), is(true));
    }

    @Test
    public void testOrderDoesNotRemoveIfNotEnoughInWarehouse() throws Exception {
        // setup
        Warehouse warehouse = mock(Warehouse.class);
        when(warehouse.getInventory(TALISKER)).thenReturn(10);
        Order order = new Order(TALISKER, 20);
        // execute
        order.fill(warehouse);
        // verify
        assertThat(order.isFilled(), is(false));
    }
}
{% endhighlight %}

### 相互作用中心のテスト(Mock使用)

{% highlight java  %}
public class OrderInteractionTest {

    private static final String TALISKER = "tarisker";

    @Test
    public void testFillingRemovesInventoryIfInStock() throws Exception {
        // setup
        Order order = new Order(TALISKER, 20);
        Warehouse warehouse = mock(Warehouse.class);
        when(warehouse.getInventory(TALISKER)).thenReturn(50);
        // execute
        order.fill(warehouse);
        // verify
        assertThat(order.isFilled(), is(true));
        verify(warehouse, times(1)).getInventory(TALISKER);
        verify(warehouse, times(1)).remove(TALISKER, 20);
    }

    @Test
    public void testFillingDoesNotRemoveIfNotEnoughInStock() throws Exception {
        // setup
        Order order = new Order(TALISKER, 20);
        Warehouse warehouse = mock(Warehouse.class);
        when(warehouse.getInventory(TALISKER)).thenReturn(10);
        // execute
        order.fill(warehouse);
        // verify
        assertThat(order.isFilled(), is(false));
        verify(warehouse, times(1)).getInventory(TALISKER);
        verify(warehouse, never()).remove(eq(TALISKER), anyInt());
    }
}
{% endhighlight %}

- 状態に着目したテストはよく見る感じ。
- 相互作用に着目したテストは、例がシンプルすぎたせいか少し良さを感じ辛かった。実際の使用例を見てみたい。例えばVisitorパターンを実装したクラスで、もしvisit順も知りたいとかいう場合は、Mockとしてテストすると簡単そう？
- また、相互作用に着目したテストは内部実装に依存したテストとなる。

<a id="anchor3"></a>

## Spring FrameworkとMockを使用したテスト

個人的にJavaを使用した開発ではフレームワークとしてSpringを使用することが多いため、SpringにおけるMock利用についても確認した。やりたいのは「Mockオブジェクトを依存性注入」。

{% highlight java  %}
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration
public class MockExampleTest {

     @Configuration
     @ComponentScan
     public static class MockExampleConfig {
         // MockitoによるMock化
         // このようにmethodではなくannotationで設定することもできる
         @Mock
         private Warehouse warehouse;

         public MockExampleConfig() {
             MockitoAnnotations.initMocks(this);
         }

         // Mock化されたBean
         @Bean
         public Warehouse warehouse() {
             return warehouse;
         }

         // MockオブジェクトがセットされたBean
         @Bean
         public WarehouseUtilizer warehouseUtilizer() {
             WarehouseUtilizer utilizer = new WarehouseUtilizer();
             utilizer.setWarehouse(warehouse());
             return utilizer;
         }
    }

    private static final String TALISKER = "tarisker";
    @Autowired Warehouse warehouse;
    @Autowired WarehouseUtilizer utilizer;
    @Autowired WarehouseAutowired autowired;

    // Mockオブジェクトによる引当成功時のテスト
    @Test
    public void testOrderIsFilledIfEnoughInWarehouse() throws Exception {
        // setup
        when(warehouse.getInventory(TALISKER)).thenReturn(50);
        Order order = new Order(TALISKER, 20);
        // execute
        order.fill(warehouse);
        // verify
        assertThat(order.isFilled(), is(true));
    }

    // Mockオブジェクトによる引当失敗時のテスト
    @Test
    public void testOrderDoesNotRemoveIfNotEnoughInWarehouse() throws Exception {
        // setup
        when(warehouse.getInventory(TALISKER)).thenReturn(10);
        Order order = new Order(TALISKER, 20);
        // execute
        order.fill(warehouse);
        // verify
        assertThat(order.isFilled(), is(false));
    }

    // ConfigurationでMockオブジェクトがセットされたBean
    @Test
    public void testUtilizer() throws Exception {
        when(warehouse.getInventory(TALISKER)).thenReturn(10);
        assertThat(utilizer.getInventory(TALISKER), is(10));
    }

    // MockオブジェクトがAutowiredされたBean
    @Test
    public void testAutowired() throws Exception {
        when(warehouse.getInventory(TALISKER)).thenReturn(10);
        assertThat(autowired.getInventory(TALISKER), is(10));
    }
}
{% endhighlight %}

- `Configuration`にconstructorを明示的に設定するという少しトリッキーな方法。だけど調べた感じ一番シンプルにやりたいことができている。
- もちろん他のBeanへのAutowired等も可能。
- この方法が使えるのはinterfaceに対してだけという記述もあったが、試す限りでは通常のクラスに対しても使用できている。

<a id="anchor4"></a>

## Apache CamelとMockを使用したテスト

あまりメジャーではないかもだけど、データ連携用のJavaフレームワークとして[Apache Camel][4]というものがあり、たまに使っている。「置かれたテキストファイルをガーッと読み込み、処理して、別の形式、プロトコルで他に投げる」とかそういうことが簡単に書ける。Apache Camelの概要は日本語の記事だと[こちら][5]がわかりやすい。ここではApache CamelでのテストにおけるMock使用法について確認した。やりたいことは「BeanコンポーネントのMock化」

{% highlight java %}
@RunWith(CamelCdiRunner.class)
@Beans(alternatives=MockBodySetterBean.class)
public class MockRouteTest {

    @EndpointInject(uri="mock:result")
    protected MockEndpoint resultEndpoint;
    @Produce(uri="direct:start")
    protected ProducerTemplate producerTemplate;

    @Named("bodySetterBean")
    public static class BodySetterBean {
        public String set() {
            return "ThisIsActualBean";
        }
    }
    @Alternative
    @Named("bodySetterBean")
    public static class MockBodySetterBean {
        public String set() {
            return "ThisIsMockBean";
        }
    }

    @Test
    public void testMockBean() throws Exception {
        // expect
        resultEndpoint.expectedBodiesReceived("ThisIsMockBean");
        // execute
        producerTemplate.sendBody("start");
        // verify
        resultEndpoint.assertIsSatisfied();
    }

    public static class MockRouteConfig extends RouteBuilder {
        @Override
        public void configure() throws Exception {
            from("direct:start")
            .bean("bodySetterBean")                            // Call bean by String
            //.bean(BodySetterBean.class)                      // Call bean by Class -> Cannot be mocked
            //.transform().simple("${bean:bodySetterBean}")    // Call bean by simple expression
            .to("mock:result")
            ;
        }
    }
}
{% endhighlight %}

- 最近作られた[CDI Testing][6]の仕組みに則って記述。
- BeanコンポーネントのMock化は結局何らかのMock用のクラスを自作するしかなさそう。
- ちなみにRoute自体のMock化なら[こちら][7]。もしくはSpring Frameworkと組み合わせてよければ、`@MockEndpointsAndSkip`と`@EndpointInject`の合わせ技で書くとすごいラク。

{% highlight java  %}
@RunWith(CamelSpringJUnit4ClassRunner.class)
@ContextConfiguration(
    loader = CamelSpringDelegatingTestContextLoader.class
)
//@MockEndpoints("direct:next|direct:prev")      // 正規表現でMock化したいRouteの指定。このRouteは実行される。
@MockEndpointsAndSkip("direct:next|direct:prev") // この場合実行されない。いわばStub的。
public class MockSpringTest {

    @Produce(uri="direct:start")
    ProducerTemplate producerTemplate;
    @EndpointInject(uri="mock:direct:next")
    MockEndpoint mockNextEndpoint;
    @EndpointInject(uri="mock:direct:prev")
    MockEndpoint mockPrevEndpoint;
   
    ...

}
{% endhighlight %}

### 使用したサンプルのリポジトリ

- [mochito-example][8]
- [spring-mock-example][9]
- [camel-mock-example][10]

[1]:http://d.hatena.ne.jp/devbankh/201002
[2]:http://qiita.com/mstssk/items/98e597c13f12746c907d
[3]:http://www.amazon.co.jp/JUnit%E5%AE%9F%E8%B7%B5%E5%85%A5%E9%96%80-~%E4%BD%93%E7%B3%BB%E7%9A%84%E3%81%AB%E5%AD%A6%E3%81%B6%E3%83%A6%E3%83%8B%E3%83%83%E3%83%88%E3%83%86%E3%82%B9%E3%83%88%E3%81%AE%E6%8A%80%E6%B3%95-WEB-PRESS-plus/dp/477415377X
[4]:http://camel.apache.org/
[5]:http://qiita.com/daikuro/items/5e22c78c0342fde04b87
[6]:http://camel.apache.org/cdi-testing.html
[7]:http://qiita.com/daikuro/items/2b9e495ae61a612f1375
[8]:https://github.com/tiqwab/example/tree/master/mockito-example
[9]:https://github.com/tiqwab/example/tree/master/spring-mock-example
[10]:https://github.com/tiqwab/example/tree/master/camel-mock-example
