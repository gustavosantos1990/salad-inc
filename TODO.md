# TODO

- ssl review
    - generate certificate with Let's Encrypt
    - replace self-signed certificate
    - identify places where ssl/tls verification is disabled and fix
    - also find potential services to enable ssl/tls
- create base Java/Spring/Maven project
- to have a way and a place to publish app swagger
- block direct pushes to master
- block direct commits to master (?)
- add spotbugs to CI/CD pipeline
- add checkstyle to CI/CD pipeline
- add PMD to CI/CD pipeline
- create web apps base template
- rename Docker manifests: <app name>-<version>-<yyyyMMddHHmmSS>
- introduce releases to projects
- review pipe variables
- to trigger email with pipe results
- share NVD database between projects
- database replication
    - check:
    .owasp-data/: found 7 matching artifact files and directories 
    .m2/: found 2379 matching artifact files and directories 
    No URL provided, cache will not be uploaded to shared cache server. Cache will be stored only locally. 
- replace at pipe vars:
    from: https://sonarqube.salad.com
    to: http://containerization.salad.com:9001
- replace at settings.xml:
    from: http://artifact-repository.salad.com:8081/repository/maven-group/
    to: https://nexus.salad.com/repository/maven-group/
- check variables existing at swarm-stack.yml and group CI/CD variables (SONAR_HOST_URL)
- fix hardcoded ip at templates (http://containerization.salad.com:9001 to http://192.168.123.22:9001)
- make sure that failed/rolled back deployments don't delete the existing stack/replicas