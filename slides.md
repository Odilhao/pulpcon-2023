---
download: true
# try also 'default' to start simple
theme: seriph
#theme: apple-basic
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
---

# Pulpcon - 2023

## Building Pulp RPMs with PEP-517 compliant packages

---
layout: intro
---

# whoami

- Odilon Sousa
  - Senior Software Engineer at Red Hat
  - Primary working with RPM packaging
    - Foreman/Katello/Pulp

---

# State of RPM's

<div grid="~ cols-2 gap-2" m="-t-2">
<div>

- 5 Releases being Supported
  - 3.16
  - 3.18
  - 3.21
  - 3.22
  - 3.28

- New Releases being packaged
  - rpm/develop
  - rpm/3.39
</div>

<div>

What's new for 2023

PEP-517 Support

COPR as build system

Nightly builds based on copr build


</div>
</div>

---

<div grid="~ cols-2 gap-2" m="-t-2">


```mermaid
%%{init: { 'logLevel': 'debug', 'theme': 'dark'} }%%
timeline
    title RPM Packaging Until 3.28 release
    section 3.18-3.22 Rpms
        3.18 : Mass rebuild for Python 3.9 : EL9 1st build
        3.21 : New release using Python 3.9 : Some packages backported into 3.18
        3.22 : Still on Python 3.9 : The madness continues with packages being backported to 3.18 and 3.21 :  Django is the primary case
    section The madness evolved
        3.28: Still on Python 3.9 : Now we have OTEL : Time to embrance PEP-517 : We still backport packages to 3.18 3.21 and 3.22 : +45 new packages for PEP-517 support
```

```mermaid
%%{init: { 'logLevel': 'debug', 'theme': 'dark'} }%%
timeline
    title RPM Packaging after 3.28
    section rpm/devolop aka nightly
        rpm/develop : Based on 3.28 : We plan to keep it update with the latest version that Katello want to consume : We will branch numered releases from here : Build are now done in COPR
        3.39 : Going to be branched from rpm/develop : Rebuilt on top of Python 3.11 : Planned for Katello 4.11
        3.40+ : Will be released after Katello 4.11 Branch
    section Next Year plan
        Automations: Auto packaging
```

</div>


---

# Why support PEP-517

<div grid="~ cols-2 gap-2" m="-t-2">



<div>

New packages don't know about setup.py <b>pyproject.toml </b> is the new kid on the block

To keep building with only setuptools we had to inject a <b>setup.py</b> in the specfile

```bash
# create a minimal setup.py, the rest will 
# be done by setuptools
printf 'from setuptools import setup\nsetup()' > setup.py

or 

# Force setuptools_scm usage for older setuptools
sed -i 's/setuptools.setup.*/setuptools.setup(use_scm_version=True)/' setup.py
```


</div>

<div>

What's necessary to support all packages that implement PEP-517 on Pulp

- Hatch build system
- Flit build system
- Poetry build system
- And a lot of circular dependency

<img border="rounded" src="https://media.giphy.com/media/55iSbtO40rIwVPVL0U/giphy-downsized-large.gif" width="150" height="200">

</div>


</div>

---

# How we built the support?

<div grid="~ cols-2 gap-2" m="-t-2">

<div>

python-tomli is the most important package to start, since it's the TOML parser, the problem it only have support for <b> pyproject.toml </b>

We had to inject a <b>setup.py</b> for the last time

<div style="width:480px"><iframe v-click class="absolute  w-80 opacity-110" allow="fullscreen" frameBorder="0" height="200" src="https://giphy.com/embed/BkRNtZvoIywTaQT2mh/video" width="480"></iframe></div>

</div>

<div v-click>

- pipdeptree and pipgrip was heavily used to solve dependecy
- 45+ new packages added
- pyproject-rpm-macros ported from Fedora to add the necessary macros

</div>

</div>
---

<div grid="~ cols-2 gap-2" m="-t-2">
<div>

Packaging structure with setup.py

```bash
%prep
set -ex
%autosetup -n %{pypi_name}-%{version}
rm -rf %{pypi_name}.egg-info

%build
set -ex
%py3_build

%install
set -ex
%py3_install

%files -n %{?scl_prefix}python%{python3_pkgversion}-%{pypi
_name}
%license LICENSE
%doc README.rst
%{python3_sitelib}/%{pypi_name}
%{python3_sitelib}/%{pypi_name}-%{version}-py%{python3_ver
sion}.egg-info
```

</div>

<div v-click>

Packaging structure with PEP-517

```bash
%prep
set -ex
%autosetup -n %{pypi_name}-%{version}

%build
set -ex
%pyproject_wheel

%install
set -ex
%pyproject_install

%files -n python%{python3_pkgversion}-%{pypi_name}
%{python3_sitelib}/opentelemetry
%{python3_sitelib}/%{pypi_name}-%{version}.dist-info/
```

</div>


</div>

---

# RPM Develop aka nightly

- Help us during branch for Katello
- Not build from one older version to one new version
  - Try to keep up to date with Pulp and branch when necessary for Pulp
- Test new releases with Katello development
  - Help identifying possible regression and blockers
- Automation! Automation! Automation
  - Create one automation that can read the list of dependencies of Pulp and check if it needs <b>setup.py</b> or <b>pyproject.toml</b>
    - Use the right tool to add or update the specfile

---

# Building packages with COPR

<div grid="~ cols-2 gap-2" m="-t-2">

<div>

Until now all RPMs got build at koji.katello.org

Maintaining our own koji demands a lot time and work

COPR Provides RHEL 8 and RHEL 9 buildroots

We can still scratch to test packages

We still use <b>obal</b> to build/scratch/lint our packages

How do we enable copr on packaging?

</div>

<div v-click>
<sub>

```yaml
copr_projects:
  vars:
    core_modules:
      - 'python39:3.9'
      - 'ruby:2.7'
    rhel_9: '9'
    rhel_8: '8'
    root_repo_url: 
  hosts:
    pulpcore-copr:
      copr_project_name: 
      copr_project_chroots:
        - name: "rhel-{{ rhel_8 }}-x86_64"
          modules: "{{ core_modules }}"
          comps_file: 
          buildroot_packages:
            - gcc-c++
            - python39-rpm-macros
          external_repos:
            - "{{ pulpcore_staging }}/rhel-{{ rhel_8 }}-x86_64"
```
</sub>
</div>
</div>
https://github.com/theforeman/obal

https://github.com/theforeman/pulpcore-packaging/blob/rpm/develop/package_manifest.yaml#L44

---

# Pulpcore Nightly 

<div grid="~ cols-2 gap-2" m="-t-2">

<div>

![Copr repo with Pulpcore Nightly](pulpcore-nightly-staging.png)

![Jenkins Pipeline with Pulpcore Nightly](pulpcore-staging-jenkins.png)

https://ci.theforeman.org/blue/organizations/jenkins/pulpcore-nightly-rpm-pipeline/

</div>


<div>

<sub>

https://copr.fedorainfracloud.org/coprs/g/theforeman/pulpcore-nightly-staging/

<br><br>


![Copr repo with Pulpcore Nightly](pulpcore-nightly-staging-scratch.png)
https://copr.fedorainfracloud.org/coprs/g/theforeman/pulpcore-nightly-staging-scratch-b26856ba-3c9d-5535-9bf7-f474701e8e4c/

</sub>

</div>

</div>

---
layout: center
---

# Questions?