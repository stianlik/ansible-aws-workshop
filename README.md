# Ansible AWS workshop

I denne workshoppen skal vi konfigurere et AWS-oppsett bestående
av:

- [Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/) med to subnet
- Jump server for sikker tilkobling til VPC
- Lastbalanserer
- Autoscaling group med to web-tjenere
- Sikkerhetsgrupper for å styre tilgang til tjenestene

## Oppgave 1: Oppsett

Installer [Ansible](http://ansible.com/) og [Boto](http://boto.cloudhackers.com/en/latest/boto_config_tut.html).

    pip install ansible
    pip install boto

Gå inn i AWS-konsollet og generer en "Access Key" på brukeren din. Gå inn i
Identity and Access Management (IAM), finn brukeren din under "Users", velg
Security Credentials, og "Create Access Key". Last ned filen, så du er sikker på å ikke miste nøklene.

Hvis du bruker din egen konto, lag en bruker under "Users" først, og gi
brukeren nødvendig tilganger (man kan godt bruke full admin i workshoppen).

Neste steg er å sette opp miljøvariabler med Access Key-en, slik at Ansible
kan logge seg på AWS.  Dette gjøres ved å sette `AWS_ACCESS_KEY_ID` og
`AWS_SECRET_ACCESS_KEY`. Dette kan enten gjøres ved å legge det inn i
`.bashrc`, ved å kjøre export-kommandoene i shellet du bruker, eller ved å konfigurere
[Boto](http://boto.cloudhackers.com/en/latest/boto_config_tut.html) med `~/.boto`.

```bash
export AWS_ACCESS_KEY_ID="<din access key>"
export AWS_SECRET_ACCESS_KEY="<din secret key>"
```

Ansible bruker [boto](http://boto.cloudhackers.com/en/latest/boto_config_tut.html) til
å kommunisere med AWS. Det er derfor også mulig å spesifisere nøklene i filen `~/.boto`.

    [Credentials]
    aws_access_key_id = <your_access_key_here>
    aws_secret_access_key = <your_secret_key_here>

## Oppgave 1: Playbook

Ansible bruker playbooks til å spesifisere konfigurasjon, utrulling og orkestrering. Opprett en ny [playbook](http://docs.ansible.com/ansible/playbooks.html) med filnavn `playbook.yml` og følgende innhold:

```yaml
---
- hosts: localhost
  tasks:
  - debug:
      msg: Da er vi igang
```

Sjekk at playbook fungerer ved å kjøre `ansible-playbook playbook.yml`.

## Oppgave 2: VPC

Vi begynner med å lage en [VPC i Ansible](http://docs.ansible.com/ansible/ec2_vpc_module.html).
Bruk `10.0.0.0/16` som `cidr_block`, og tag med `ansible-workshop` som `Name`.

```yaml
- hosts: localhost
  tasks:
  - name: create vpc
    ec2_vpc:
        region: eu-central-1
        state: present
        cidr_block: 10.0.0.0/16
        resource_tags:
          Name: ansible-workshop
    register: vpc
```

For å sjekke hvilke endringer Ansible har tenkt til å gjøre, kjør [`ansible-playbook
--check playbook.yml`](http://docs.ansible.com/ansible/playbooks_checkmode.html). Hvis alt ser riktig
ut, bruk [`ansible-playbook playbook.yml`](http://docs.ansible.com/ansible/playbooks.html)
for å utfør endringene.

Instansene må også ha mulighet til å koble seg til Internett. For å få tilgang til Internett trenger vi en [Internet
Gateway](http://docs.ansible.com/ansible/ec2_vpc_igw_module.html).
Sett opp en slike gateway. Tips: Internet gateway kan spesifiseres direkte
i `ec2_vpc`-modulen.

Ved å refere til
[VPC-en](http://docs.ansible.com/ansible/ec2_vpc_subnet_module.html) definert
tidligere, kan man hente ut ID-en, og bruke den når man definerer opp
sikkerhetsgruppene. Id-en kan hentes ut med synaksen `"{{ aws_vpc.vpc_id }}"` ([Jinja2 template engine](http://jinja.pocoo.org/)) etter at den
er registrert med `register: aws_vpc`. Se [Variables](http://docs.ansible.com/ansible/playbooks_variables.html#registered-variables)
for mer informasjon.

## Oppgave 3: Sett opp subnet

Nå skal vi sette opp to [subnet i
VPC-en](http://docs.ansible.com/ansible/ec2_vpc_subnet_module.htm) vår.
La hvert subnet være i hver sin `availability_zone`. La et
subnet bruke `cidr_block` `10.0.1.0/24` og det andre bruke `10.0.2.0/24`.

For at maskiner på subnettet skal kunne koble seg til Internett må man lage en [rutingtabell](http://docs.ansible.com/ansible/ec2_vpc_route_table_module.html), og assosiere tabellen med subnettet. Opprett en rute `0.0.0.0/0` til internet gatewayen.

## Oppgave 4

Nå skal vi sette opp to webservere og en lastbalanserer til å serve innholdet
ut på Internett. Istedenfor å kjøre opp serverene manuelt, så lager vi en
autoscalinggroup for serverene.

### Oppgave 4.1: Lastbalanserer

For at lastbalanserer skal være tilgjengelig på nett, trenger den en [sikkerhetsgruppe](http://docs.ansible.com/ansible/ec2_group_module.html) med
åpning inn på port `80` med TCP mot alle IP-adresser (`0.0.0.0/0`).
Husk å spesifisere VPC-ID på den nye sikkerhetsgruppen, siden gruppen er tilordnet en VPC.

```yaml
- proto: tcp
  from_port: 80
  to_port: 80
  cidr_ip: 0.0.0.0/0
```

Vi begynner med å lage en
[lastbalanserer](http://docs.ansible.com/ansible/ec2_elb_lb_module.html) for
webapplikasjonen. Lastbalansereren skal lytte på port `80`, og kjøre helsesjekk
mot applikasjonen på port `80` på URL-en `/healthcheck.txt`. Applikasjonen
kjører også på port `80`. Man må også definere hvilke subnet som
lastbalansereren skal være koblet på. Bruk variabelreferanser til de subnettene
vi definerte i den tidligere oppgaven. Huske å definere
`listeners`, `health_check` og `security_group_names`.

Kjør `ansible-playbook playbook.yml -vvv` for å informasjon om resultatet. Prøv å gå på URL-en i browseren. Siden vi ikke har noen app som kjører skal du
få en helt blank side, som returnerer 503 Service Unavailable.

### Oppgave 4.2: Jump server

Siden vi har lyst til å logge inn på serverene for å sjekke at alt kom riktig
opp, må vi lage et [Key
Pair](https://www.terraform.io/docs/providers/aws/r/key_pair.html) med kommandoen
`ssh-keygen -f id_rsa`. Innholdet i `id_rsa.pub` skal da
legges inn i feltet `key_material` i [ec2](http://docs.ansible.com/ansible/ec2_module.html)-modulen. Man kan enten kopiere
innholdet manuelt, eller bruke
[with_file](http://docs.ansible.com/ansible/playbooks_loops.html#looping-over-files)-attributten.

Et tips kan være å kjøre `ssh-add id_rsa` for å slippe å
spesifisere nøkkel når du skal prøve å logge deg inn på serverene senere.

Vi må også sette opp en [sikkerhetsgruppe](http://docs.ansible.com/ansible/ec2_group_module.html) for å
kontrollere hvilke porter som er tilgjengelig. Maskinene vi skal sette opp
trenger port `22` (slik at vi kan bruke SSH inn) tilgjengelig fra overalt
(`0.0.0.0/0`), og port `80` (slik at lastbalansereren får tilgang til webappen)
tilgjengelig for lastbalansereren. Webappen skal kun være tilgjengelig for
lastbalanseren, og ikke hele verden.  Man kan bruke en security group som
source, istedenfor en IP-adresse i AWS. Tips er derfor å bruke
[`group_name`](http://docs.ansible.com/ansible/ec2_group_module.html) for å angi tilgang fra
lastbalansereren uten å måtte oppgi IP.

Husk å spesifisere VPC-ID på den nye sikkerhetsgruppen.

### Oppgave 4.3: Launch configuration

An autoscaling group (AG) trenger en [Launch
Configuration](http://docs.ansible.com/ansible/ec2_lc_module.html)
for å kunne vite hva slags type instancer som skal startes. `image_id` finner
man ved å logge inn på AWS-konsollet, og begynne å starte en EC2 instans i
regionen man bruker. Hver region har sine egne AMI-ider.

Hvis man legger til et script i `user_data`, så blir dette kjørt når instansen
kommer opp. Les inn innholdet fra `startup.sh` ved hjelp av
[`with_file`](http://docs.ansible.com/ansible/playbooks_loops.html#looping-over-files)-funksjonen, og putt det inn i `user_data`-feltet.

*Merk at launch configuration ikke kan endres. For å endre en eksisterende konfigurasjon
må det opprettes en ny launch configuration.*

### Oppgave 4.4: Auto-scaling group

Helt til slutt skal vi lage en [auto-scaling
group](http://docs.ansible.com/ansible/ec2_asg_module.html)
som skal kjøre opp maskinene våre. Vi skal ha to instanser basert på launch
configuration fra forrige oppgave, fordelt på de to subnettene fra tidligere
(via `vpc_zone_identifier`).  Husk også å spesifisere hvilken `load_balancer`
som maskinene skal meldes inn i.

Når du kjører `ansible-playbook playbook.yml` så auto-scaling groupen blir laget, så skal du
AWS Console se at det starter opp to maskiner, og du vil etterhvert få opp en
side i browseren hvis du går inn på URL-en til lastbalansereren.

Prøv gjerne også å logge deg inn på serverene via SSH. IP-adressene finner du i
AWS Console.

## Ferdig?

Antagelig ikke helt. Fjern alt du har laget via AWS Console, eller ved å bruk
av en playbook der du bruker `state`-attributten til å fjerne tjenester.
