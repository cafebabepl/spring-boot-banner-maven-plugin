# Implementacja własnej wtyczki do Mavena

### Wstęp
W moim przekonaniu nawet nie wypada pisać czym jest Maven, ale jakiś wstęp trzeba napisać.
(Musi być wstęp, rozwinięcie i zakończenie. Dodatkowo przyjmiemy formę *1 per. pluralis 
indicativus*).

Jedyne co możemy popełnić to przypomnienie, że jedną z podstawowych 
koncepcji Mavena są tzw. cykle życia (*build lifecycle*). Maven definiuje trzy podstawowe 
cykle: `clean` (*project cleaning*), `default` (*project deployment*) i `site` (*creation project's site documentation*). 
Każdy z tych cykli zdefniowany jest przez listę faz (*phase*), które wykonywane są w 
określonej kolejności. Dla cyklu domyślnego będą to m.in.: `validate`, `compile`, `test`, 
`package`, `verify`, `install` i `deploy` (pełna lista faz jest dużo dłuższa).
Wywołanie polecenia `mvn install` spowoduje wykonanie faz domyślnego cyklu 
począwszy od pierwszej (`validate`) aż do wywołanej (`install`) włącznie. Każda faza 
z kolei składa się z wykonania celów (*goal*) właściwych... wtyczek.

W ten sposób mamy precyzyjnie i powtarzalnie zdefiniowany zestaw kroków, które zostaną 
wykonane podczas budowania naszej aplikacji. Nic nie stoi jednak na przeszkodzie, aby 
w ten zdefiniowany cykl wpleść mniej lub bardziej wyuzdaną fukncjonalność, która jest nam 
niezbędna.

Każdy kto chociaż raz uruchomił aplikację z wykorzystaniem Spring Boot z domyślnymi 
ustawieniami zobaczył na konsoli dumny baner w stylu:

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.3.RELEASE)
```

Oczywiście w łatwy sposób możemy go wyłączyć lub podłączyć własny. Spróbujemy jednak 
tworzenie naszego własnego logo wpiąć w cykl budowania naszej aplikacji.

### Projekt (in. rozwinięcie)
Projekt własnej wtyczki do Mavena utworzymy i będziemy budować oczywiście w... Mavenie.

```bash
mvn archetype:generate -B \
    -DarchetypeArtifactId=maven-archetype-quickstart \
    -DgroupId=com.asseco.maven.plugin -DartifactId=spring-boot-banner-maven-plugin \
    -Dversion=1.0.0 \
    -Dpackage=com.asseco.maven.plugin.banner
```

Podobno są tylko [dwie trudne rzeczy w informatyce](https://martinfowler.com/bliki/TwoHardThings.html): *cache invalidation and naming things* 
(chociaż do mnie bardziej przemawia wersja "rozproszona": *There are only two hard problems in distributed systems:  2. Exactly-once delivery 1. Guaranteed order of messages 2. Exactly-once delivery*).
Pierwszego mechanizmu nie będziemy używać, drugi problem rozwiązali za nas twórcy Mavena. 
Zgodnie z konwencją nazwa naszego pluginu powinna mieć format `<yourplugin>-maven-plugin`, nazwa w stylu `maven-<yourplugin>-plugin` 
zarezerwowana jest dla oficjalnych wtyczek Apache Maven.

Po utworzeniu projektu, w pierwszym kroku, dodajemy zależności `maven-plugin-api`,
który zapewni nam dostęp do klasy `AbstractMojo` oraz `maven-plugin-annotations`, który 
umożliwi wykorzystanie adnotacji `@Mojo`.

W zerowym kroku (czyli jeszcze przed krokiem pierwszym) musimy zmienić domyślny parametr z 
`<packaging>jar</packaging>` na `<packaging>maven-plugin</packaging>`.
Bez tej zmiany w trakcie budowania wtyczki nie zostanie utworzony tzw. deskryptor wtyczki 
i próba jej uruchomienia zakończy się błędem: 
  
```
[ERROR] Failed to parse plugin descriptor for com.asseco.maven.plugin:spring-boot-banner-maven-plugin:1.0.0
(c:\Users\Wlodzimierz.Kozlowsk\.m2\repository\com\asseco\maven\plugin\spring-boot-banner-maven-plugin\1.0.0\spring-boot-banner-maven-plugin-1.0.0.jar):
No plugin descriptor found at META-INF/maven/plugin.xml -> [Help 1]
```

POM powinien w tym momencie wyglądać mniej więcej w ten sposób:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.asseco.maven.plugin</groupId>
    <artifactId>spring-boot-banner-maven-plugin</artifactId>
    <packaging>maven-plugin</packaging>
    <version>1.0.0</version>
    <name>spring-boot-banner-maven-plugin</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.maven</groupId>
            <artifactId>maven-plugin-api</artifactId>
            <version>3.6.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.maven.plugin-tools</groupId>
            <artifactId>maven-plugin-annotations</artifactId>
            <version>3.6.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```

Czym jest Mojo?
MOJO = POJO (ang. *Plain Old Java Object*) + Maven, a samo słowo oznacza:
> ...a small bag worn by a person under the clothes (also known as a mojo hand). Such bags were thought to have supernatural powers, such as protecting from evil, bringing good luck, etc.

Uzbrojeni w te nadprzyrodzone moce tworzymy klasę:

```java
@Mojo(name = "hello")
public class SpringBootBannerMojo extends AbstractMojo {

    @Override
    public void execute() throws MojoExecutionException, MojoFailureException {
        getLog().info("Szumią jodły na gór szczycie...");
    }
    
}
```
Adnotacja `@Mojo` definiuje nam nazwę celu (*goal name*), metoda `execute`, jak łatwo 
się domyślić, wszystko to, co ma być wykonane w ramach uruchomienia naszego celu. *To* 
to krótkie i pojemne słowo i nie stawia nam zbyt wielu ograniczeń.

Zbudujemy zatem naszą wtyczkę `mvn clean install` i spróbujemy ją uruchomić, na początek z linii poleceń, 
poprzez wywołanie polecenia w formacie `mvn groupId:artifactId:version:goal`
czyli w naszym przypadku `mvn com.asseco.maven.plugin:spring-boot-banner-maven-plugin:1.0.0:hello`.

Chwilę później (tutaj chwila trwała 0,264 s) powinniśmy uzyskać efekt naszego wywołania:

```
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building spring-boot-banner-maven-plugin 1.0.0
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- spring-boot-banner-maven-plugin:1.0.0:hello (default-cli) @ spring-boot-banner-maven-plugin ---
[INFO] Szumią jodły na gór szczycie...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 0.264 s
[INFO] Finished at: 2019-03-20T16:32:54+01:00
[INFO] Final Memory: 15M/368M
[INFO] ------------------------------------------------------------------------
```

Zbudowaliśmy działającą wtyczkę! Dostarcza już jakąś funkcjonalność, jest wysokiej jakości (wydaje się, że nie zawiera zbyt wielu błędów) i... 
można ją zarejestrować jako dzieło dla 50% KUP.
Możnaby na tym poprzestać, ale dla zbudowania przewagi nad konkurencją spróbujemy jeszcze ją nieco 
rozbudować.

Na początek dodajmy kilka parametrów wywołania. Wystarczy dodać pole i oznaczyć je adnotacją `@Parameter`.
Sam parametr możemy... sparametryzować. Domyślnie nazwa parametru `name` jest taka jak 
nazwa pola.

```java
    @Parameter(property = "spring-boot-banner.text", required = true)
    private String text;
```

Jeśli pole oznaczymy jako wymagane `required = true`, próba uruchomienia 
wtyczki bez przekazania wartości tego parametru zakończy się błędem:

```
[ERROR] Failed to execute goal com.asseco.maven.plugin:spring-boot-banner-maven-plugin:1.0.0:generate (default) on project aums-mdm-readings-api:
The parameters 'text' for goal com.asseco.maven.plugin:spring-boot-banner-maven-plugin:1.0.0:generate are missing or invalid -> [Help 1]
```

Każdy parametr może mieć ustaloną wartość domyślną, zarówno "na sztywno":

```java
    @Parameter(property = "spring-boot-banner.font", defaultValue = "standard")
    private String font;
```
jak i można użyć wyrażenia, którego wartość jest dostępna w naszym POM, odwołując się np. do wyjściowego 
katalogu kompilacji:

```java
    @Parameter(property = "spring-boot-banner.directory", defaultValue = "${project.build.outputDirectory}")
    private File directory;
```

Każdemu parametrowi możemy nadać `property` dzięki czemu możemy ustawić jego wartość nie 
tylko w sekcji `configuration` samego pluginu:
```xml
    <configuration>
        <text>cafebabe</text>
    </configuration>
```
ale zarówno we właściwościach projektu POM, np.:

```xml
    <properties>
        <java.version>1.8</java.version>
        <spring-boot-banner.font>block</spring-boot-banner.font>
    </properties>
```
jak i wywołać z linii poleceń.

Warto zwrócić uwagę na jakiś sensowny prefiks tych właściwości, żeby nie popaść w konflikt z 
pozostałymi ustawieniami.

Parametry `text` i `font` nie wymagają chyba żadnego komentarza - to odpowiednio tekst 
i czcionka FIGlet, które będą użyte do wygenerowania naszego baneru.

W aplikacji Spring Boot domyślną wartością parametru `spring.banner.location` jest `classpath:banner.txt`. Zatem 
jeśli chcemy podmienić domyślny baner na niestandardowe logo wystarczy własny plik o 
nazwie `banner.txt` umieścić w katalogu `src/main/resources` i zbudować aplikację. 
Tej domyślnej konfiguracji z `application.properties` odpowiadają domyślne wartości parametrów 
`directory` i `filename` naszej wtyczki. Gdybyśmy z jakiś względów chcieli zmienić tę domyślną konfigurację 
aplikacji powinniśmy ją także uwzględnić w konfiguracji wtyczki.

Komentarza wymaga jedynie ostatni parametr tj. `request`. W dobie mikroserwisów nie będziemy 
generować pliku ani samodzielnie, ani z wykorzystaniem zewnętrznych bibliotek, ale pozwolimy 
sobie na wywołanie zdalnego API. Jedną z dostępnych usług jest https://devops.datenkollektiv.de/banner.txt/index.html 
i dla niej skonfigurowany jest domyślny szablon wywołania usługi. Jeśli zajdzie potrzeba 
zmiany bo, na przykład, napiszemy "lepszą swoją" albo znajdziemy "lepszą inną" zmiana 
konfiguracji parametru pozwoli nam wykorzystać inną usługę bez zmiany wtyczki.

Pozostaje nam to wszystko poskładać w jedną całość. Uzupełnijmy zależności w POM o:

```xml
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5.7</version>
        </dependency>
```

Pełna implementacja naszej wtyczki wygląda teraz tak:

```java
package com.asseco.maven.plugin.banner;

import java.io.File;
import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyManagementException;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;

import org.apache.commons.lang3.StringUtils;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.conn.ssl.TrustAllStrategy;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.ssl.SSLContextBuilder;
import org.apache.http.util.EntityUtils;
import org.apache.maven.plugin.AbstractMojo;
import org.apache.maven.plugin.MojoExecutionException;
import org.apache.maven.plugin.MojoFailureException;
import org.apache.maven.plugins.annotations.LifecyclePhase;
import org.apache.maven.plugins.annotations.Mojo;
import org.apache.maven.plugins.annotations.Parameter;

@Mojo(name = "generate", defaultPhase = LifecyclePhase.GENERATE_RESOURCES, threadSafe = true, requiresOnline = true)
public class SpringBootBannerMojo extends AbstractMojo {

    private static final String UTF_8 = StandardCharsets.UTF_8.name();

    @Parameter(property = "spring-boot-banner.text", required = true)
    private String text;

    @Parameter(property = "spring-boot-banner.font", defaultValue = "standard")
    private String font;

    @Parameter(property = "spring-boot-banner.filename", defaultValue = "banner.txt")
    private String filename;

    @Parameter(property = "spring-boot-banner.directory", defaultValue = "${project.build.outputDirectory}")
    private File directory;

    @Parameter(property = "spring-boot-banner.request", defaultValue = "https://devops.datenkollektiv.de/renderBannerTxt?text={text}&font={font}")
    private String request;

    private static String replaceGetParam(final String uri, final String search, final String replacement) {
        try {
            return StringUtils.replace(uri, search, URLEncoder.encode(replacement, UTF_8));
        } catch (UnsupportedEncodingException ex) {
            throw new RuntimeException(ex);
        }
    }

    private String getBannerQuery(final String uri) throws MojoExecutionException {
        /*
         * The price of checked exceptions is an Open/Closed Principle1 violation.
         * If you throw a checked exception from a method in your code and the catch is three levels above, 
         * you must declare that exception in the signature of each method between you and the catch.
         * This means that a change at a low level of the software can force signature changes on many higher levels.
         * The changed modules must be rebuilt and redeployed, even though nothing they care about changed.
         *
         * Robert C. Martin, "Clean Code. A Handbook of Agile Software Craftsmanship", p. 107
         */
        getLog().info(uri);
        try (CloseableHttpClient httpClient = HttpClients.custom()
                .setSSLContext(new SSLContextBuilder().loadTrustMaterial(null, TrustAllStrategy.INSTANCE).build())
                .build()) {

            HttpGet httpGet = new HttpGet(uri);
            CloseableHttpResponse response = httpClient.execute(httpGet);
            return EntityUtils.toString(response.getEntity());
        } catch (KeyManagementException | NoSuchAlgorithmException | KeyStoreException | IOException ex) {
            throw new MojoExecutionException("Oops. The banner could not be generated.", ex);
        }
    }

    private void saveBannerCommand(final String banner) throws MojoExecutionException {
        directory.mkdirs();
        Path path = directory.toPath().resolve(filename);
        try {
            Files.write(path, banner.getBytes(UTF_8));
        } catch (IOException ex) {
            throw new MojoExecutionException("Oops. The file could not be saved.", ex);
        }
    }

    @Override
    public void execute() throws MojoExecutionException, MojoFailureException {
        // Skoro jodły już szumią,
        getLog().info("Szumią jodły na gór szczycie...");

        // pozostaje nam przygotować URI,
        String uri = request;
        uri = replaceGetParam(uri, "{text}", text);
        uri = replaceGetParam(uri, "{font}", font);

        // wywołać RESTa, pobrać wynik...
        final String banner = getBannerQuery(uri);

        // ... i zapisać do pliku. 
        saveBannerCommand(banner);
    }

}
```

Po zbudowaniu naszej wtyczki `mvn clean install` możemy ją wywołać z linii poleceń, np.

```
mvn com.asseco.maven.plugin:spring-boot-banner-maven-plugin:1.0.0:generate \
    -Dspring-boot-banner.text=Assecoo \
    -Dspring-boot-banner.directory=C:/temp
```

lub użyć w jakimś "spring-boot'owym" projekcie (albo jakimkolwiek innym).

W Maven mamy dostępne wtyczki służące do budowania (cykl `default`, konfigurowane w 
znaczniku `<build>`) oraz raportowania (cykl `site`, konfigurowane w znaczniku `<reporting>`). 
Nasza wtyczka należy do tej pierwszej grupy więc jej przykładowe użycie wygląda jak poniżej:

```xml
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                </plugin>
                <plugin>
                    <groupId>com.asseco.maven.plugin</groupId>
                    <artifactId>spring-boot-banner-maven-plugin</artifactId>
                    <version>1.0.0</version>
                    <configuration>
                        <text>cafebabe</text>
                        <font>block</font>
                    </configuration>
                    <executions>
                        <execution>
                            <phase>generate-resources</phase>
                            <goals>
                                <goal>generate</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
```

W trakcie wykonania fazy `generate-resources` zostanie wywołany cel `generate` z ustawionymi 
parametrami `text` i `font`. Po zbudowaniu i uruchomieniu aplikacji:

```bash
mvn clean install spring-boot:run
```
osiągamy pełen sukces. Marketing będzie zadowolony! 

```
[INFO] <<< spring-boot-maven-plugin:2.1.3.RELEASE:run (default-cli) < test-compile @ aums-mdm-readings-api <<<
[INFO]
[INFO] --- spring-boot-maven-plugin:2.1.3.RELEASE:run (default-cli) @ aums-mdm-readings-api ---

                           _|_|              _|                    _|
   _|_|_|     _|_|_|     _|         _|_|     _|_|_|       _|_|_|   _|_|_|       _|_|
 _|         _|    _|   _|_|_|_|   _|_|_|_|   _|    _|   _|    _|   _|    _|   _|_|_|_|
 _|         _|    _|     _|       _|         _|    _|   _|    _|   _|    _|   _|
   _|_|_|     _|_|_|     _|         _|_|_|   _|_|_|       _|_|_|   _|_|_|       _|_|_|


2019-03-22 10:00:40.013  INFO 11488 --- [           main] com.asseco.mdm.readings.api.Application  : Starting Application on APS00065640 with PID 11488 
```

### Podsumowanie (in. zakończenie)
Przedstawiona tutaj implementacja wtyczki jest oczywiście tylko pretekstem, ale mam nadzieję, 
że pokazuje prosty a równocześnie potężny mechanizm budowania własnych wtyczek dla Mavena. Mechanizm, 
który w naszym cyklu budowania i wdrażania aplikacji pozwoli dodać własne kroki abyśmy 
mogli automatyzować wszystko to co tylko można zautomatyzować bez potrzeby sprawdzonego 
ale żmudnego użycia klawisza F5.


#### /src/main/resources
- https://github.com/wkozi/spring-boot-banner-maven-plugin
- [Apache Maven Project. Introduction to the Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Apache Maven Project. Plugin Developers Centre](https://maven.apache.org/plugin-developers/index.html)
- [Spring Boot Reference Guide. Customizing the Banner](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#boot-features-banner)
- ["Szumią jodły na gór szczycie" i Wołodymyr Kuźmenko](https://youtu.be/t0PXRig8Ou0?t=32)

#### @author
**Włodzimierz Kozłowski**  
wlodzimierz.kozlowski@asseco.pl  
Starszy Projektant  
Pion Energetyki i Gazownictwa  
Asseco Poland S.A.
