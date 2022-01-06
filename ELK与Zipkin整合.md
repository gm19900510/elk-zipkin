# 一、需求描述
在分布式系统中，日志跟踪是一件很令程序员头疼的问题，在遇到生产问题时，如果是多节点需要打开多节点服务器去跟踪问题，如果下游也是多节点且调用多个服务，那就更麻烦，再者，如果没有分布式链路，在生产日志飞速滑动的情况下，很难找出问题。

所以，分布式系统中很有必要搭建一套分布式日志系统，笔者采用了市面成熟的解决方案ELK+Zipkin解决
# 二、组件依赖
[利用ELK 搭建日志分析系统（一）—— 组件介绍](https://blog.csdn.net/ctwy291314/article/details/122101398)

[利用ELK 搭建日志分析系统（二）—— 部署安装](https://blog.csdn.net/ctwy291314/article/details/122107428)

[利用ELK 搭建日志分析系统（三）—— 安全认证](https://blog.csdn.net/ctwy291314/article/details/122200344)


[Zipkin 服务链路追踪](https://blog.csdn.net/ctwy291314/article/details/122351357)

# 三、整体效果
![在这里插入图片描述](https://img-blog.csdnimg.cn/6c0c8f08856849c4a2b77719bbecb2a0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2b71a2bcc1f3471a90687d8d8f37868e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
# 四、Gateway 改造 
通过`GlobalFilter` 打印路由日志，`ReqTraceFilter.java`

```java
import java.net.URI;
import java.util.List;
import java.util.Map;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.util.MultiValueMap;
import org.springframework.web.server.ServerWebExchange;
import com.ruoyi.common.core.utils.StringUtils;
import lombok.extern.slf4j.Slf4j;
import reactor.core.publisher.Mono;

@Component
@Slf4j
public class ReqTraceFilter implements GlobalFilter, Ordered {

	private static final String REQUEST_PREFIX = "Request Info [ ";

	private static final String REQUEST_TAIL = " ]";

	private final static String[] VALIDATE_URL = new String[] { "/auth/login", "/auth/register" };

	@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		ServerHttpRequest request = exchange.getRequest();
		if (StringUtils.containsAnyIgnoreCase(request.getURI().getPath(), VALIDATE_URL)) {
			return chain.filter(exchange);
		}
		StringBuilder normalMsg = new StringBuilder();
		HttpMethod method = request.getMethod();
		URI uri = request.getURI();
		HttpHeaders headers = request.getHeaders();
		String url = "";
		Object requestParams = null;
		if (HttpMethod.GET.equals(method)) {
			MultiValueMap<String, String> queryParams = request.getQueryParams();
			StringBuilder builder = new StringBuilder();
			for (Map.Entry<String, List<String>> entry : queryParams.entrySet()) {
				builder.append(entry.getKey()).append("=").append(StringUtils.join(entry.getValue(), "&")).append("&");
			}
			if (builder.length() > 0) {
				requestParams = builder.toString().substring(0, builder.toString().length() - 1);
				url = uri.getPath() + "?" + requestParams;
			} else {
				requestParams = builder.toString();
				url = uri.getPath();
			}

		} else {
			url = uri.getPath();
		}

		normalMsg.append(REQUEST_PREFIX);
		normalMsg.append("HEADER=").append(headers);
		normalMsg.append(";METHOD=").append(method.name());
		normalMsg.append(";URL=").append(url);
		normalMsg.append(REQUEST_TAIL);

		log.info(normalMsg.toString());
		return chain.filter(exchange.mutate().request(request.mutate().build()).build());

	}

	@Override
	public int getOrder() {
		return -2;
	}

}
```
