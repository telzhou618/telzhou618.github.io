# 【k8s学习笔记】 Kubesphere CI/CD流谁线自动化部署


&lt;!--more--&gt;

## 准备一个 springboot 项目

项目如下，是一个简单的springboot项目

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506222151.png)

定义一个简单接口, 返回 helloworld

```java
package com.example.demo;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
@RestController
public class DemoApplication {

	@GetMapping(&#34;/&#34;)
	public String home() {
		return &#34;hello world!!!&#34;;
	}

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}

```

单元测试类 DemoApplicationTests.java 

```java
package com.example.demo;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.junit4.SpringRunner;
import static org.assertj.core.api.Assertions.assertThat;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class DemoApplicationTests {

	@Test
	public void contextLoads() {
	}

	@Autowired
	private TestRestTemplate restTemplate;

	@Test
	public void homeResponse() {
		String body = this.restTemplate.getForObject(&#34;/&#34;, String.class);
		assertThat(body).isNotBlank();
	}
}

```



## 开启devops功能

登录 KubeSphere ，找到定制资源，搜索config

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506222844.png)

点击 ClusterConfiguration，然后点击三个点编辑YAML

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506223036.png)

编辑配置，将devops 的 enabled 改为true

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506223307.png) 

## 流水线部署

创建流水线项目

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506230107.png)

进入项目创建流水线

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506230210.png)

编辑流水线

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506230257.png)

全局代理选择 maven

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506230346.png)

第一步拉取代码

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506230638.png)

第二部单元测试

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506230512.png)

第三部打包并构建镜像

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506230715.png)

部署到指定k8s集群

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506230750.png)

搞定！

运行流水线

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506231244.png)

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506231313.png)

运行日志

![](https://raw.gitmirror.com/telzhou618/images/main/img03/20240506231341.png)

## 所需配置文件

完整的 jenkinsfile 文件内容

```jenkins
pipeline {
  agent {
    node {
      label &#39;maven&#39;
    }

  }
  stages {
    stage(&#39;拉取代码&#39;) {
      agent none
      steps {
        git(url: &#39;http://192.168.1.4/root/java-web-demo.git&#39;, credentialsId: &#39;gitlab&#39;, branch: &#39;master&#39;, changelog: true, poll: false)
      }
    }

    stage(&#39;单元测试&#39;) {
      agent none
      steps {
        container(&#39;maven&#39;) {
          sh &#39;mvn clean test&#39;
        }

      }
    }

    stage(&#39;构建镜像&#39;) {
      agent none
      steps {
        container(&#39;maven&#39;) {
          sh &#39;mvn -Dmaven.test.skip=true clean package&#39;
          sh &#39;docker build -f deploy/Dockerfile -t $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER .&#39;
          withCredentials([usernamePassword(credentialsId: &#39;dockerhub-id&#39;, passwordVariable: &#39;DOCKER_PASSWORD&#39;, usernameVariable: &#39;DOCKER_USERNAME&#39;)]) {
            sh &#39;echo &#34;$DOCKER_PASSWORD&#34; | docker login $REGISTRY -u &#34;$DOCKER_USERNAME&#34; --password-stdin&#39;
            sh &#39;docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER&#39;
          }

        }

      }
    }

    stage(&#39;发布到test环境&#39;) {
      agent none
      steps {
        container(&#39;maven&#39;) {
          withCredentials([kubeconfigContent(credentialsId: &#39;demo-kubeconfig&#39;, variable: &#39;KUBECONFIG_CONTENT&#39;)]) {
            sh &#39;&#39;&#39;mkdir ~/.kube
echo &#34;$KUBECONFIG_CONTENT&#34; &gt; ~/.kube/config
envsubst &lt; deploy/k8s-deploy-test.yaml | kubectl apply -f -&#39;&#39;&#39;
          }

        }

      }
    }

  }
  environment {
    DOCKER_CREDENTIAL_ID = &#39;dockerhub-id&#39;
    GITHUB_CREDENTIAL_ID = &#39;github-id&#39;
    KUBECONFIG_CREDENTIAL_ID = &#39;demo-kubeconfig&#39;
    REGISTRY = &#39;docker.io&#39;
    DOCKERHUB_NAMESPACE = &#39;telzhou618&#39;
    GITHUB_ACCOUNT = &#39;kubesphere&#39;
    APP_NAME = &#39;java-web-demo-test&#39;
  }
}
```

Dockerfile 文件

```dockerfile
FROM java:8u92-jre-alpine
MAINTAINER &#34;telzhou618&#34;
ADD target/java-web-demo-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT [&#34;java&#34;, &#34;-jar&#34;, &#34;/app.jar&#34;]
```

k8s-deploy.yaml 文件, 一个 Deployment, 一个service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: java-web-demo-test
  name: java-web-demo-test
  namespace: bbs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-web-demo-test
  template:
    metadata:
      labels:
        app: java-web-demo-test
    spec:
      containers:
        - name: java-web-demo-test
          image: $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER
          ports:
            - containerPort: 8080
              protocol: TCP
          resources:
            limits:
              cpu: &#34;1&#34;
            requests:
              cpu: 500m
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: java-web-demo-test
  name: java-web-demo-test
  namespace: bbs
spec:
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
      nodePort: 30963
  selector:
    app: java-web-demo-test
  type: NodePort

```


---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/a9d96f7/  

