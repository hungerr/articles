# Maven常用插件

## exec-maven-plugin

### 目标

- `exec:exec` execute programs and Java programs in a separate process.
- `exec:java` execute Java programs in the same VM.

### POM配置

需要`maven`最低版本`3.2.5`，`JDK`最低版本`1.8`
```XML
<project>
  ...
  <build>
    <!-- To define the plugin version in your parent POM -->
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>org.codehaus.mojo</groupId>
          <artifactId>exec-maven-plugin</artifactId>
          <version>3.1.0</version>
        </plugin>
        ...
      </plugins>
    </pluginManagement>
    <!-- To use the plugin goals in your POM or parent POM -->
    <plugins>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>3.1.0</version>
      </plugin>
      ...
    </plugins>
  </build>
  ...
</project>
```

### Exec goal

`mvn exec:exec -Dexec.executable="maven" [-Dexec.workingdir="/tmp"] -Dexec.args="-X myproject:dist"`

或者通过POM配置：
```XML
<project>
  ...
  <build>
    <plugins>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>3.1.0</version>
        <executions>
          <execution>
            ...
            <goals>
              <goal>exec</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <executable>maven</executable>
          <!-- optional -->
          <workingDirectory>/tmp</workingDirectory>
          <arguments>
            <argument>-X</argument>
            <argument>myproject:dist</argument>
            ...
          </arguments>
          <environmentVariables>
            <LANG>en_US</LANG>
          </environmentVariables>
        </configuration>
      </plugin>
    </plugins>
  </build>
   ...
</project>
```

### Java goal

`mvn exec:java -Dexec.mainClass="com.example.Main" [-Dexec.args="argument1"] ...`

或者通过POM配置：
```XML
<project>
  ...
  <build>
    <plugins>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>3.1.0</version>
        <executions>
          <execution>
            ...
            <goals>
              <goal>java</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <mainClass>com.example.Main</mainClass>
          <arguments>
            <argument>argument1</argument>
            ...
          </arguments>
          <systemProperties>
            <systemProperty>
              <key>myproperty</key>
              <value>myvalue</value>
            </systemProperty>
            ...
          </systemProperties>
        </configuration>
      </plugin>
    </plugins>
  </build>
   ...
</project>
```