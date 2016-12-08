# Ansible AWS workshop

## Oppgave 1: Oppsett

Installer [Ansible](http://ansible.com/) og [Boto](http://boto.cloudhackers.com/en/latest/boto_config_tut.html).

    pip install ansible
    pip install boto

Gå inn i AWS-konsollet, og generer en "Access Key" på brukeren din. Gå inn i
Identity and Access Management (IAM), finn brukeren din under "Users", velg
Security Credentials, og "Create Access Key". Last ned fila, så du er sikker på
å ikke miste nøklene.

Hvis du bruker din egen konto, lag en bruker under "Users" først, og gi
brukeren nødvendig tilganger (man kan godt bruke full admin i workshoppen).

Neste steg er å sette opp miljøvariabler med Access Key-en, slik at Ansible
kan logge seg på AWS.  Dette gjøres ved å sette `AWS_ACCESS_KEY_ID` og
`AWS_SECRET_ACCESS_KEY`. Dette kan enten gjøres ved å legge det inn i
`.bashrc`, ved å kjøre export-kommandoene i shellet du bruker, eller ved konfigurere
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

## Oppgave 2: VPC

Vi begynner med å lage en [VPC i Ansible](http://docs.ansible.com/ansible/ec2_vpc_module.html).
Bruk `10.0.0.0/16` som `cidr_block`, og tag med `ansible-workshop` som `Name`.

For å sjekke hvilke endringer Ansible har tenkt til å gjøre, kjør [`ansible-playbook
--check playbook.yml`](http://docs.ansible.com/ansible/playbooks_checkmode.html). Hvis alt ser riktig
ut, bruk [`ansible-playbook playbook.yml`](http://docs.ansible.com/ansible/playbooks.html)
for å utfør endringene.

Instansene må også ha mulighet til å koble seg til Internett. For å få tilgang
til Internett trenger vi en [Internet
Gateway](http://docs.ansible.com/ansible/ec2_vpc_igw_module.html).
Sett opp en slike gateway. Tips: Se over mulige parametre for `ec2_vpc`-modulen før
du starter med `ec2_vpc_igw`.

Ved å refere til
[VPC-en](http://docs.ansible.com/ansible/ec2_vpc_subnet_module.html) definert
tidligere, kan man hente ut ID-en, og bruke den når man definerer opp
subnettene. Id-en kan hentes ut med synaksen `{{ aws_vpc.vpc_id }}` etter at den
er registrert med `register: aws_vpc`. Se [Variables](http://docs.ansible.com/ansible/playbooks_variables.html#registered-variables)
for mer informasjon.

## Oppgave 3: Sett opp subnet

Nå skal vi sette opp to [subnet i
VPC-en](http://docs.ansible.com/ansible/ec2_vpc_subnet_module.htm) vår.
La vært subnet være i hver sin `availability_zone`. La et
subnet bruke `cidr_block` `10.0.1.0/24` og det andre bruke `10.0.2.0/24`.

For at maskiner på subnettet skal kunne koble seg til internett trenger man å
lage en [route
table](http://docs.ansible.com/ansible/ec2_vpc_route_table_module.html), og så
assosiere tabellen med subnettet.
Så man skal route `0.0.0.0/0` til internet gatewayen.

## Oppgave 4

Nå skal vi sette opp to webservere og en lastbalanserer til å serve innholdet
ut på internet. Istedenfor å kjøre opp serverene manuelt, så lager vi en
autoscalinggroup for serverene. Prøve å lag all infrastrukturen i denne
oppgaven inn i en [Ansible role](http://docs.ansible.com/ansible/playbooks_roles.html), ved å sende inn all
informasjonen du trenger via inputs til modulen.

### Oppgave 4.1: Lastbalanserer

For at lastbalanserer skal være tilgjengelig på nett, trenger den en [Security
Group](https://www.terraform.io/docs/providers/aws/r/security_group.html) med
åpning inn (`ingress`) på port `80` med TCP mot alle IP-adress (`0.0.0.0/0`).
Husk at du i Terraform også må legge til en regel med full åpning ut fra
lastbalansereren også (`egress`). Husk å spesifisere VPC-ID på den nye Security
Group-en, siden SG-er er tilordnet en VPC.

```terraform
    # Åpent ut mot alt
    egress {
        from_port = 0
        to_port = 0
        protocol = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
```

Vi begynner med å lage en
[lastbalanserer](https://www.terraform.io/docs/providers/aws/r/elb.html) for
webapplikasjonen. Lastbalansereren skal lytt på port `80`, og kjøre helsesjekk
mot applikasjonen på port `80` på URL-en `/healthcheck.txt`. Applikasjonen
kjører også på port `80`. Man må også definere hvilke subnet som
lastbalansereren skal være koblet på. Bruk variabelreferanser til de subnettene
vi definerte i den tidligere oppgaven.  Lister defineres slik i Terraform:
`["${modul.navn.variabel}", "${modul.navn.variabel}"]`. Huske å definer opp en
`listener`- og en `health_check`-blokk.

Husk å spesifisere `security_groups` når du oppretter lastbalansereren.

Legg til adressen (DNS-navnet) til lastbalansereren som en
[output](https://www.terraform.io/docs/configuration/outputs.html) både fra
modulen, og fra hovedoppskriften. Når du kjører `terraform apply` skal du få ut URL-en til lastbalanseren helt til slutt.

Prøv å gå på URL-en i browseren. Siden vi ikke har noen app som kjører skal du
få en helt blank side, som returnerer 503 Service Unavailable.

### Oppgave 4.2: Sikkerhet

Siden vi har lyst til å logge inn på serverene for å sjekke at alt kom riktig
opp, må vi lage et [Key
Pair](https://www.terraform.io/docs/providers/aws/r/key_pair.html) først.
Terraform støtter ikke å generere et nøkkelpar, så vi kan bruke kommandoen
`ssh-keygen -f terraform-workshop`. Innholdet i `terraform-workshop.pub` skal da
legges inn i feltet `public_key` i terraform-fila vår. Man kan enten kopiere
innholdet manuelt, eller bruke
[`file()`](https://www.terraform.io/docs/configuration/interpolation.html#element_list_index_)-funksjonen

Et tips kan være å kjøre `ssh-add terraform-workshop` for å slippe å
spesifisere nøkkel når du skal prøve å logge deg inn på serverene senere.

Vi må også sette opp en [Security
Group](https://www.terraform.io/docs/providers/aws/r/security_group.html) for å
kontrollere hvilke porter som er tilgjengelig. Maskinene vi skal sette opp
trenger port `22` (så vi kan bruke SSH inn) tilgjengelig fra overalt
(`0.0.0.0/0`), og port `80` (så lastbalansereren får tilgang til webappen)
tilgjengelig for lastbalansereren. Webappen skal kun være tilgjengelig for
lastbalanseren, og ikke hele verden.  Man kan bruke en security group som
source, istedenfor en IP-adresse i AWS. Tips er derfor å bruke
[`source_security_group_id`](https://www.terraform.io/docs/providers/aws/r/elb.html#source_security_group_id)
som er en variabel fra lastbalansereren, for å angi tilgang fra
lastbalansereren uten å måtte oppgi IP.

Husk å spesifisere VPC-ID på den nye Security Group-en, siden SG-er er
tilordnet en VPC.


### Oppgave 4.3: Launch configuration

An autoscaling group (AG) trenger en [Launch
Configuration](https://www.terraform.io/docs/providers/aws/r/launch_configuration.html)
for å kunne vite hva slags type instancer som skal startes. `image_id` finner
man ved å logge inn på AWS-konsollet, og begynne å starte en EC2 instans i
regionen man bruker. Hver region har sine egne AMI-ider.

Hvis man legger til et script i `user_data`, så blir dette kjørt når instansen
kommer opp. Les inn innholdet fra `startup.sh` ved hjelp av
[`file()`](https://www.terraform.io/docs/configuration/interpolation.html#element_list_index_)-funksjonen, og putt det inn i `user_data`-feltet.

Det kan også være en god ide å sette
[create_bebfore_destroy](https://www.terraform.io/docs/configuration/resources.html#lifecycle)
til `true`, siden man ikke kan endre Launch Configuration. Da vil Terraform
lage en ny LC, oppdatere AG, og så lette den gamle LC igjen helt til slutt. På
den måten så får man ikke feil når man endrer Launch Configuration.
**NB!** Ikke glem å legg til keypair og Security group.


### Oppgave 4.4: Auto-scaling group

Helt til slutt skal vi lage en [auto-scaling
group](https://www.terraform.io/docs/providers/aws/r/autoscaling_group.html)
som skal kjøre opp maskinene våre. Vi skal ha to instanser basert på launch
configuration fra forrige oppgave, fordelt på de to subnettene fra tidligere
(via `vpc_zone_identifier`).  Husk også å spesifisere hvilken `load_balancer`
som maskinene skal meldes inn i.

Når du kjører `terraform apply` så auto-scaling groupen blir laget, så skal du
AWS Console se at det starter opp to maskiner, og du vil etterhvert få opp en
side i browseren hvis du går inn på URL-en til lastbalansereren.

Prøv gjerne også å logge deg inn på serverene via SSH. IP-adressene finner du i
AWS Console.

## Ferdig?

Antagelig ikke helt. Fjern alt du har laget:

```bash
terraform destroy
```
