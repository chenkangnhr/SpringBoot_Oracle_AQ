1 Oracle 数据库代码
1.1 用户AQ相关权限的设置
SYS用户以DBA用户登录Oracle，将下列权限赋给SCOTT用户
GRANT EXECUTE ON SYS.DBMS_AQ to SCOTT;
GRANT EXECUTE ON SYS.DBMS_AQADM to SCOTT;
GRANT EXECUTE ON SYS.DBMS_AQ_BQVIEW to SCOTT;
GRANT EXECUTE ON SYS.DBMS_AQIN to SCOTT;
GRANT EXECUTE ON SYS.DBMS_JOB to SCOTT;
1.2 创建队列相关对象并启动队列
--创建queue table
begin
  sys.dbms_aqadm.create_queue_table(queue_table        => 'QUEUE_MAP_MESSAGE_TABLE',
                                    queue_payload_type => 'sys.aq$_jms_map_message',
                                    sort_list          => 'ENQ_TIME',
                                    compatible         => '10.0.0',
                                    primary_instance   => 0,
                                    secondary_instance => 0);
end;
--创建队列
begin
  sys.dbms_aqadm.create_queue(queue_name     => 'QUEUE_MAP_MESSAGE',
                              queue_table    => 'QUEUE_MAP_MESSAGE_TABLE',
                              queue_type     => sys.dbms_aqadm.normal_queue,
                              max_retries    => 5,
                              retry_delay    => 0,
                              retention_time => 0);
end;
--启动队列
begin
  -- 启动队列
  sys.dbms_aqadm.start_queue(queue_name => 'QUEUE_MAP_MESSAGE');

  /*-- 暂停队列
  sys.dbms_aqadm.STOP_QUEUE(
      queue_name => 'QUEUE_MAP_MESSAGE'
  );
  
  -- 删除队列
  sys.dbms_aqadm.DROP_QUEUE(
      queue_name => 'QUEUE_MAP_MESSAGE'
  );
  
  -- 删除对列表
  sys.dbms_aqadm.DROP_QUEUE_TABLE(
     queue_table => 'QUEUE_MAP_MESSAGE_TABLE'
  );
  */
end;
1.3 创建入队存储过程
CREATE OR REPLACE PROCEDURE pkg_enqueue_map_message(message_id      VARCHAR2,
                                                    message_content VARCHAR2) as
  id                 pls_integer;
  agent              sys.aq$_agent := sys.aq$_agent(' ', null, 0);
  l_message          sys.aq$_jms_map_message;
  enqueue_options    dbms_aq.enqueue_options_t;
  message_properties dbms_aq.message_properties_t;
  msgid              raw(16);
  java_exp exception;
  pragma EXCEPTION_INIT(java_exp, -24197);
BEGIN
  -- Consturct a empty map l_message object
  l_message := sys.aq$_jms_map_message.construct;
  -- Shows how to set the JMS header
  l_message.set_replyto(agent);
  l_message.set_type('tkaqpet1');
  l_message.set_userid('jmsuser');
  l_message.set_appid('plsql_enq');
  l_message.set_groupid('st');
  l_message.set_groupseq(1);
  -- Shows how to set JMS user properties
  l_message.set_string_property('color', 'RED');
  l_message.set_int_property('year', 1999);
  l_message.set_float_property('price', 16999.99);
  l_message.set_long_property('mileage', 300000);
  l_message.set_boolean_property('import', True);
  l_message.set_byte_property('password', -127);
  -- Shows how to populate the l_message payload of aq$_jms_map_message
  -- Passing -1 reserve a new slot within the l_message store of sys.aq$_jms_map_message.
  -- The maximum number of sys.aq$_jms_map_message type of messges to be operated at
  -- the same time within a session is 20. Calling clean_body function with parameter -1
  -- might result a ORA-24199 error if the messages currently operated is already 20.
  -- The user is responsible to call clean or clean_all function to clean up l_message store.
  id := l_message.clear_body(-1);
  -- Write data into the l_message paylaod. These functions are analogy of JMS JAVA api 's.
  -- See the document for detail.
  -- Set a byte entry in map l_message payload
  l_message.set_string(id, 'message_id', message_id);
  l_message.set_string(id, 'message_content', message_content);

  l_message.flush(id);
  -- Use either clean_all or clean to clean up the l_message store when the user
  -- do not plan to do paylaod population on this l_message anymore
  sys.aq$_jms_map_message.clean_all();
  --l_message.clean(id);
  --Enqueue this l_message into AQ queue using DBMS_AQ package
  dbms_aq.enqueue(queue_name         => 'QUEUE_MAP_MESSAGE',
                  enqueue_options    => enqueue_options,
                  message_properties => message_properties,
                  payload            => l_message,
                  msgid              => msgid);
  commit;
end pkg_enqueue_map_message;

2 Java代码
2.1 OracleDataSourceFactory
Oracle连接配置
package com.king.oracleaq.config;


import oracle.jdbc.pool.OracleDataSource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.sql.SQLException;

@Configuration
public class OracleDataSourceFactory {

    /** 数据库用户名 */
    @Value("${spring.datasource.druid.username}")
    public String userName;

    /** 数据库密码 */
    @Value("${spring.datasource.druid.password}")
    public String password;

    /** 数据库地址url */
    @Value("${spring.datasource.druid.url}")
    public String url;

    @Bean
    public OracleDataSource getOracleDataSource() throws SQLException {
        OracleDataSource oracleDataSource = new OracleDataSource();
        oracleDataSource.setURL(url);
        oracleDataSource.setUser(userName);
        oracleDataSource.setPassword(password);
        return oracleDataSource;
    }

}

2.2 OracleAQQueueConnectionFactory
Oracle Queue连接工厂
package com.king.oracleaq.config;


import oracle.jdbc.pool.OracleDataSource;
import oracle.jms.AQjmsFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.jms.JMSException;
import javax.jms.QueueConnectionFactory;

@Configuration
public class OracleAQQueueConnectionFactory {
    @Bean
    public QueueConnectionFactory getAQjmsFactory(OracleDataSource oracleDataSource) throws JMSException {
        QueueConnectionFactory queueConnectionFactory = AQjmsFactory.getQueueConnectionFactory(oracleDataSource);
        return queueConnectionFactory;
    }

}
2.3 OracleAqQueueFactory
获取Oracle Queue
package com.king.oracleaq.config;

import lombok.extern.slf4j.Slf4j;
import oracle.jms.AQjmsSession;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import javax.jms.Destination;
import javax.jms.Queue;
import javax.jms.QueueConnectionFactory;
import javax.jms.Session;

@Component
@Slf4j
public class OracleAqQueueFactory {

    @Value("${queue.name}")
    private String oracleQueueName;
    @Value("${spring.datasource.druid.username}")
    private String oracleQueueUser;
    private final QueueConnectionFactory queueConnectionFactory;

    public OracleAqQueueFactory(QueueConnectionFactory queueConnectionFactory) {
        this.queueConnectionFactory = queueConnectionFactory;
    }

    @Value("${queue.name}")
    private String queueName;


    @Bean("oracleAQQueue")
    public Destination getDestination() throws Exception {
        return getObject();
    }


    protected Queue getObject() throws Exception {
        final AQjmsSession session = (AQjmsSession)queueConnectionFactory
                .createConnection()
                .createSession(true, Session.SESSION_TRANSACTED);
        Queue queue = session.getQueue(oracleQueueUser, oracleQueueName);
        log.info("QueueName: {}", queue.getQueueName());
        return queue;
    }
}

2.4 OracleAQJmsTemplate
package com.king.oracleaq.config;


import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jms.core.JmsTemplate;

import javax.jms.Destination;
import javax.jms.QueueConnectionFactory;

@Configuration
public class OracleAQJmsTemplate {
    private final QueueConnectionFactory queueConnectionFactory;
    private final Destination oracleAQQueue;

    public OracleAQJmsTemplate(QueueConnectionFactory queueConnectionFactory,
                               Destination oracleAQQueue) {
        this.queueConnectionFactory = queueConnectionFactory;
        this.oracleAQQueue = oracleAQQueue;
    }
    @Bean
    public JmsTemplate getJmsTemplate() throws Exception {
        JmsTemplate jmsTemplate = new JmsTemplate();
        jmsTemplate.setConnectionFactory(queueConnectionFactory);
        jmsTemplate.setDefaultDestination(oracleAQQueue);
        return jmsTemplate;
    }
}

2.5 MessageListenerContainer
创建MessageConsumer
package com.king.oracleaq.bean;


import oracle.jms.AQjmsSession;
import org.springframework.jms.listener.DefaultMessageListenerContainer;

import javax.jms.Destination;
import javax.jms.JMSException;
import javax.jms.MessageConsumer;
import javax.jms.Session;


public class MessageListenerContainer extends DefaultMessageListenerContainer {

    @Override
    protected MessageConsumer createConsumer(Session session, Destination destination) throws JMSException {
        MessageConsumer messageConsumer = ((AQjmsSession) session).createConsumer(destination);
        return messageConsumer;
    }

}

2.6 OracleAQMessageListenerContainer
package com.king.oracleaq.config;


import com.king.oracleaq.bean.MessageListenerContainer;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import javax.jms.Destination;
import javax.jms.QueueConnectionFactory;

@Component
public class OracleAQMessageListenerContainer {

    private QueueConnectionFactory queueConnectionFactory;
    private Destination oracleAQQueueName;

    public OracleAQMessageListenerContainer(QueueConnectionFactory queueConnectionFactory, Destination oracleAQQueueName, OracleAQMessageConsumer oracleAqMessageConsumer) {
        this.queueConnectionFactory = queueConnectionFactory;
        this.oracleAQQueueName = oracleAQQueueName;
        this.oracleAqMessageConsumer = oracleAqMessageConsumer;
    }

    private OracleAQMessageConsumer oracleAqMessageConsumer;


    @Bean("messageListenerContainer")
    public MessageListenerContainer getOracleAQMessageListenerContainer() {
        MessageListenerContainer messageListenerContainer = new MessageListenerContainer();
        messageListenerContainer.setConnectionFactory(queueConnectionFactory);
        messageListenerContainer.setDestination(oracleAQQueueName);
        messageListenerContainer.setMessageListener(oracleAqMessageConsumer);
        messageListenerContainer.setSessionTransacted(true);
        return messageListenerContainer;
    }
}

2.7  OracleAQMessageConsumer
消息消费者
package com.king.oracleaq.config;


import lombok.extern.slf4j.Slf4j;
import oracle.jms.AQjmsMapMessage;
import org.springframework.jms.listener.SessionAwareMessageListener;
import org.springframework.stereotype.Component;

import javax.jms.JMSException;
import javax.jms.Session;

@Component
@Slf4j
public class OracleAQMessageConsumer implements SessionAwareMessageListener<AQjmsMapMessage> {
    @Override
    public void onMessage(AQjmsMapMessage aQjmsMapMessage, Session session) {
        try {
            log.info("JMSMessageID:{}" , aQjmsMapMessage.getJMSMessageID());
            log.info("message_id:{}" , aQjmsMapMessage.getString("message_id"));
            log.info("message_content:{}", aQjmsMapMessage.getString("message_content"));
        } catch (JMSException e) {
            throw new RuntimeException(e);
        }
    }
}

2.8 OracleAqMessageApplication启动类
package com.king.oracleaq;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OracleAqMessageApplication {

    public static void main(String[] args) {
        SpringApplication.run(OracleAqMessageApplication.class, args);
    }

}
2.9 application.yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: oracle.jdbc.driver.OracleDriver
    druid:
      url: jdbc:oracle:thin:@127.0.0.1:1521:DEV01
      username: SCOTT
      password: 123456
server:
  port: 8080
queue:
  enable: true
  name: QUEUE_MAP_MESSAGE

2.10 pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.12.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.king.oracleaq</groupId>
    <artifactId>oracleaq</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>OracleAQMessageApplication</name>
    <description>OracleAQMessageApplication</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.2.15</version>
        </dependency>

        <dependency>
            <groupId>javax.transaction</groupId>
            <artifactId>jta</artifactId>
            <version>1.1</version>
        </dependency>


        <dependency>
            <groupId>com.oracle.database.jdbc</groupId>
            <artifactId>ojdbc8</artifactId>
            <version>19.3.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.3</version>
        </dependency>
        <dependency>
            <groupId>com.oracle.database.messaging</groupId>
            <artifactId>aqapi</artifactId>
            <version>19.3.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>org.springframework.jms</artifactId>
            <version>3.2.1.RELEASE</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>

3 测试CASE
begin
  -- Call the procedure
  pkg_enqueue_map_message(message_id      => '00001',
                          message_content => 'Hello, Oracle AQ: This is my first message');
end;
