# Java Code Style Configuration (`style-config`)

A shared Maven package that centralizes Eclipse Java formatting and Checkstyle validation rules across multiple projects.

## Contents

This package includes:
- `eclipse-formatter.xml`: Eclipse Java formatter configuration (K&R style, 4-space indentation, parameter descriptions on the same line).
- `checkstyle.xml`: Checkstyle validation rules (requires braces on end of line, indentation verification, etc.).
- `pre-commit`: A cross-platform Git pre-commit hook that automatically formats files and validates style before commits are completed.

## Build and Install Locally

To build the package and install it to your local Maven repository (`~/.m2`), run:

```bash
mvn clean install
```

## How to Consume

To use these shared formatting rules in another Maven project, configure the following plugins in your project's `pom.xml`:

### 1. Unpack Formatter Configuration and Git Hook
Since the Spotless plugin requires a physical file path on the disk, and Git requires the hook script in the repository's `.githooks/` directory, use the `maven-dependency-plugin` to unpack both files from this JAR during the `initialize` build phase:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>3.6.0</version>
    <executions>
        <execution>
            <id>unpack-formatter-config</id>
            <phase>initialize</phase>
            <goals>
                <goal>unpack</goal>
            </goals>
            <configuration>
                <artifactItems>
                    <artifactItem>
                        <groupId>com.example</groupId>
                        <artifactId>style-config</artifactId>
                        <version>1.0.0</version>
                        <type>jar</type>
                        <overWrite>true</overWrite>
                        <includes>eclipse-formatter.xml</includes>
                        <outputDirectory>${project.build.directory}/spotless</outputDirectory>
                    </artifactItem>
                    <artifactItem>
                        <groupId>com.example</groupId>
                        <artifactId>style-config</artifactId>
                        <version>1.0.0</version>
                        <type>jar</type>
                        <overWrite>true</overWrite>
                        <includes>pre-commit</includes>
                        <outputDirectory>${project.basedir}/.githooks</outputDirectory>
                    </artifactItem>
                </artifactItems>
            </configuration>
        </execution>
    </executions>
</plugin>
```

*(Note: For nested projects, change the hook `outputDirectory` path to `${project.basedir}/../.githooks`.)*

### 2. Configure Git Hooks and Executable Permissions
Configure Git to use the `.githooks/` directory and ensure the pre-commit script has executable permissions (critical on Unix/macOS) during build initialization:

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
        <execution>
            <id>configure-git-hooks</id>
            <phase>initialize</phase>
            <goals>
                <goal>exec</goal>
            </goals>
            <configuration>
                <executable>git</executable>
                <!-- For nested projects, add: <workingDirectory>${project.basedir}/..</workingDirectory> -->
                <arguments>
                    <argument>config</argument>
                    <argument>core.hooksPath</argument>
                    <argument>.githooks</argument>
                </arguments>
            </configuration>
        </execution>
    </executions>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
        <execution>
            <id>make-hook-executable</id>
            <phase>initialize</phase>
            <goals>
                <goal>run</goal>
            </goals>
            <configuration>
                <target>
                    <chmod file="${project.basedir}/.githooks/pre-commit" perm="ugo+rx"/>
                    <!-- For nested projects, use: file="${project.basedir}/../.githooks/pre-commit" -->
                </target>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 3. Configure Spotless Formatter
Point Spotless to the extracted configuration file:

```xml
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <version>2.38.0</version>
    <configuration>
        <java>
            <eclipse>
                <file>${project.build.directory}/spotless/eclipse-formatter.xml</file>
            </eclipse>
        </java>
        <includes>
            <include>src/main/java/**/*.java</include>
            <include>src/test/java/**/*.java</include>
        </includes>
    </configuration>
</plugin>
```

### 4. Configure Checkstyle Validation
Add this artifact as a dependency to the `maven-checkstyle-plugin` so it can load the `checkstyle.xml` rules directly from the classpath:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.2.1</version>
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>style-config</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
    <configuration>
        <configLocation>checkstyle.xml</configLocation>
        <encoding>${project.build.sourceEncoding}</encoding>
        <consoleOutput>true</consoleOutput>
        <failsOnError>true</failsOnError>
    </configuration>
    <executions>
        <execution>
            <id>verify</id>
            <phase>verify</phase>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 5. Configure IDE (VS Code)
To make VS Code format code exactly like Spotless, update your project's `.vscode/settings.json`:

```json
{
    "java.configuration.updateBuildConfiguration": "automatic",
    "java.format.settings.url": "${workspaceRoot}/target/spotless/eclipse-formatter.xml",
    "editor.formatOnSave": true
}
```
