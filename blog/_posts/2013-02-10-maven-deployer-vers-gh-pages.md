---
layout: post
title: Maven, déployer sur Github simplement
---

Quand on fait de petits modules maven, parfois c'est pratique de les déployer quelque part pour les utiliser dans d'autres projets. La question est : où les déployer ? 
En local ? Ce n'est pas très pratique si on utilise plusieurs postes et surtout ça ne facilite pas la tâche à ceux qui souhaiteraient réutiliser le module en question. 
Sur son serveur perso ? Oui on en a tous un, mais niveau qualité de service et surtout niveau disponibilité c'est beaucoup moins garanti. 

J'ai vu quelques tuto pour déployer ses artefacts sur Github. Le problème c'est qu'ils s'intégraient dans un cycle de release automatique avec le maven-release-plugin. J'ai cherché un peu et j'ai trouvré la solution suivante.

Le principe est le même que dans les différents articles que j'ai lu : la différence c'est qu'on pousse dans la branche gh-pages avec le site-maven-plugin de com.github.github au lieu de le faire à la main. 

1. déployer dans un répertoire local / configurer le maven-deploy-plugin pour déployer dans `${basedir}/target/<<repository_to_deploy>>/`
2. pousser ce qui a été généré dans la branche gh-pages / configurer le maven-site-plugin pour pousser `${basedir/target/<<repository_to_deploy>>/`

On ajoute dans le pom.xml : 

    <project>
        <distributionManagement>
            <repository>
                <id>github-releases</id>
                <url>http://<<github_account>>.github.com/<<github_project_name>>/repository/releases</url>
            </repository>
            <snapshotRepository>
                <id>github-snapshots</id>
                <url>http://<<github_account>>.github.com/<<github_project_name>>/repository/snapshots</url>
            </snapshotRepository>
        </distributionManagement>
        <build>
            <pluginManagement>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-deploy-plugin</artifactId>
                        <version>2.7</version>
                        <configuration>
                            <altDeploymentRepository>local::default::file://${project.build.directory}/gh-pages/repository/snapshots</altDeploymentRepository>
                            <uniqueVersion>false</uniqueVersion>
                        </configuration>
                    </plugin>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-dependency-plugin</artifactId>
                        <version>2.6</version>
                    </plugin>
                    <plugin>
                        <groupId>com.github.github</groupId>
                        <artifactId>site-maven-plugin</artifactId>
                        <version>0.7</version>
                        <configuration>
                            <message>deploying artefacts for version ${project.version}</message>
                            <merge>true</merge>
                            <outputDirectory>${project.build.directory}/gh-pages</outputDirectory>
                            <noJekyll>true</noJekyll>
                        </configuration>
                    </plugin>
                </plugins>
            </pluginManagement>
        </build>
        <profiles>
            <profile>
                <!-- profil pour déployer les releases dans un sous répertoire "releases" -->
                <id>release</id>
                <activation>
                    <property>
                        <name>performRelease</name>
                    </property>
                </activation>
                <build>
                    <plugins>
                        <plugin>
                            <groupId>org.apache.maven.plugins</groupId>
                            <artifactId>maven-deploy-plugin</artifactId>
                            <configuration>
                                <altDeploymentRepository>local::default::file://${project.build.directory}/gh-pages/repository/releases</altDeploymentRepository>
                                <updateReleaseInfo>true</updateReleaseInfo>
                            </configuration>
                        </plugin>
                    </plugins>
                </build>
            </profile>
        </profiles>
    </project>

Avec cette configuration on peut exécuter `mvn deploy` pour déployer et les artefacts sont disponibles via :

    http://<<github_account>>.github.com/<<github_project_name>>/repository/snapshots/
    http://<<github_account>>.github.com/<<github_project_name>>/repository/releases/

Notez que [Github pages](http://pages.github.com/) ne liste pas le contenu des répertoires. On ne peut pas parcourir le dépot, mais les fichiers sont bien accessibles. 

Vous pouvez jeter un oeil à [dudie-maven-plugin](https://github.com/dudie/dudie-maven-plugin/) si ce n'est pas encore assez clair. 

* [http://dudie.github.com/dudie-maven-plugin/repository/releases/fr/dudie/maven/plugin/maven-metadata.xml](http://dudie.github.com/dudie-maven-plugin/repository/releases/fr/dudie/maven/plugin/maven-metadata.xml)
* [http://dudie.github.com/dudie-maven-plugin/repository/releases/fr/dudie/maven/plugin/dudie-maven-plugin/1.0/dudie-maven-plugin-1.0.pom](http://dudie.github.com/dudie-maven-plugin/repository/releases/fr/dudie/maven/plugin/dudie-maven-plugin/1.0/dudie-maven-plugin-1.0.pom)
