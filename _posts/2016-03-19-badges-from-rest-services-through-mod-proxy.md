---
title: Badges from REST services through mod_proxy
updated: 2015-09-06 18:00
---

I see many projects on [GitHub](https://github.com/), which use badges to show information about build status, code coverage and so forth.

![build_passing](https://img.shields.io/badge/build-passing-brightgreen.svg)
![build_passing](https://img.shields.io/badge/tests-42%20%2F%2042-brightgreen.svg)
![build_passing](https://img.shields.io/badge/coverage-100%-brightgreen.svg)
![build_passing](https://img.shields.io/badge/dependencies-up%20to%20date-brightgreen.svg)

I really like the idea, because it gives a quick overview of the current state. If all badges are green, the project is probably fine. 
If not the corresponding issues should be addressed. 

In a local environment many tools provide information through a REST API. 
For example [Jenkins](https://jenkins-ci.org/), [SonarQube](http://www.sonarqube.org/) or [Atlassian JIRA](https://www.atlassian.com/software/jira). 
There are some badge plug-ins available for Jenkins [build status](https://wiki.jenkins-ci.org/display/JENKINS/Embeddable+Build+Status+Plugin) or SonarQube [quality gate](https://github.com/QualInsight/qualinsight-plugins-sonarqube-badges).
Since I am interested in more information (e.g. code coverage) I decided to get these information directly from their REST API and convert them into a badge.

### Get information from REST services
Unfortunately it is not possible to access the services directly from within the browser (XMLHttpRequest with jQuery), because of the [same-origin policy](https://en.wikipedia.org/wiki/Same-origin_policy).

```javascript
    var url = 'http://sonar.example.com:9000/api/resources?resource=prj&metrics=coverage';
    $.getJSON(url, function (response) {
        // will not work
    }
```
The easiest way I could think of to avoid this was the usage of a reverse proxy, which adds the Access-Control-Allow-Origin header (CORS). This is a sample config for two services. 

```docker
FROM ubuntu 
MAINTAINER simdevmon

ENV PORT_JIRA 8080
ENV URL_JIRA http://jira.example.com:8080/
ENV PORT_SONAR 9000
ENV URL_SONAR http://sonar.example.com:9000/

ENV PORT_CONFIG /etc/apache2/ports.conf
ENV SITE_CONFIG /etc/apache2/sites-enabled/000-default.conf

RUN \
 apt-get update && \ 
 apt-get install -y \ 
 apache2 \
 libapache2-mod-proxy-html \
 libxml2-dev

RUN \ 
 a2enmod proxy proxy_http && \
 a2enmod headers && \
 service apache2 restart && \
 service apache2 stop

RUN \ 
 echo '<VirtualHost *:'${PORT_JIRA}'>' > "${SITE_CONFIG}" && \   
 echo '    Header set Access-Control-Allow-Origin "*"' >> "${SITE_CONFIG}" && \
 echo '    ProxyPass / '${URL_JIRA} >> "${SITE_CONFIG}" && \
 echo '    ProxyPassReverse / '${URL_JIRA} >> "${SITE_CONFIG}" && \
 echo '</VirtualHost>' >> "${SITE_CONFIG}" && \
 echo '<VirtualHost *:'${PORT_SONAR}'>' >> "${SITE_CONFIG}" && \   
 echo '    Header set Access-Control-Allow-Origin "*"' >> "${SITE_CONFIG}" && \
 echo '    ProxyPass / '${URL_SONAR} >> "${SITE_CONFIG}" && \
 echo '    ProxyPassReverse / '${URL_SONAR} >> "${SITE_CONFIG}" && \
 echo '</VirtualHost>' >> "${SITE_CONFIG}" 
 
RUN \
 echo 'Listen '${PORT_JIRA} > "${PORT_CONFIG}" && \
 echo 'Listen '${PORT_SONAR} >> "${PORT_CONFIG}"
 
RUN \
 cat ${SITE_CONFIG} && \
 cat ${PORT_CONFIG} && \
 apachectl configtest

CMD /bin/bash -c "service apache2 start; tail -f /dev/null"
```

The docker image can be build and executed like this.

```shell
docker build -t badge_proxy .
docker run -p 8080:8080 -p 9000:9000 badge_proxy
```
Now it is possible to use XMLHttpRequests from within the browser to all required services. 

### Convert information into badges

[Shields.io](https://github.com/badges/shields) provides a great way to generate badges. 
They also provide a docker image, which is described in the [installation guide](https://github.com/badges/shields/blob/master/INSTALL.md). 

In this example I will just use the online service to generate the badge.

```html
<script type="text/javascript">
function coverageColor(coverage) {
	var color = 'brightgreen';
	if (coverage < 90) {
		color = 'yellow';
	}
	if (coverage < 70) {
		color = 'red';
	}
	return color;
}

var url = 'http://sonar.example.com:9000/api/resources?resource=prj&metrics=coverage';
var baseImage = 'https://img.shields.io/badge/';
$.getJSON(url, function (response) {
	var coverage;
	var metrics = response[0].msr;
	for (var i = 0; i < metrics.length; i++) {
		if (metrics[i].key === 'coverage') {
			coverage = metrics[i].val;
		}
	}
	document.getElementById('coverage').src = baseImage +
			'coverage-' + coverage + ' %-' + coverageColor(coverage) + '.svg';

}).error(function (jqXHR, textStatus, errorThrown) {
	document.getElementById('coverage').src = baseImage +
			'coverage-N%20/%20A-red.svg';
});
</script>
<a href="http://sonar.example.com:9000">
    <img id="coverage" src="#" alt=""/>
</a>
```

To see if a service is reachable is even easier.

```html
<script type="text/javascript">
function isServiceUp(url, name, imgId) {
    var baseImage = 'https://img.shields.io/badge/' + name;
    $.ajaxSetup({cache: false});
    $.ajax({
        url: url,
        method: "GET",
        success: function (data, status, jqxhr) {
            document.getElementById(imgId).src = baseImage + '-up-brightgreen.svg';
        },
        error: function (jqxhr, status, error) {
            document.getElementById(imgId).src = baseImage + '-down-red.svg';
        }
    });
}
isServiceUp('http://sonar.example.com:9000/api/system/status', 'sonarqube', 'sonarup');
</script>
<a href="http://sonar.example.com:9000">
    <img id="sonarup" src="#" alt=""/>
</a>   
```