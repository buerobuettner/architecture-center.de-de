---
title: Ausführen von Linux-VMs für eine n-schichtige Anwendung in Azure
description: Vorgehensweise zum Ausführen von Linux-VMs in einer n-schichtigen Architektur in Microsoft Azure
author: MikeWasson
ms.date: 11/22/2017
pnp.series.title: Linux VM workloads
pnp.series.next: multi-region-application
pnp.series.prev: multi-vm
ms.openlocfilehash: 8d3e6e5124a0abb27a3c72e1ecbd52a1a1da2a33
ms.sourcegitcommit: e67b751f230792bba917754d67789a20810dc76b
ms.translationtype: HT
ms.contentlocale: de-DE
ms.lasthandoff: 04/06/2018
---
# <a name="run-linux-vms-for-an-n-tier-application"></a>Ausführen von Linux-VMs für eine n-schichtige Anwendung

Anhand dieser Referenzarchitektur werden einige bewährte Methoden für die Ausführung virtueller Linux-Computer (VMs) für eine n-schichtige Anwendung veranschaulicht. [**So stellen Sie diese Lösung bereit**.](#deploy-the-solution)  

![[0]][0]

*Laden Sie eine [Visio-Datei][visio-download] mit dieser Architektur herunter.*

## <a name="architecture"></a>Architecture

Es gibt viele Möglichkeiten für die Implementierung einer n-schichtigen Architektur. Das Diagramm zeigt eine typische 3-schichtige Webanwendung. Diese Architektur baut auf den Informationen unter [Run load-balanced VMs for scalability and availability][multi-vm] (Ausführen von VMs mit Lastenausgleich zur Erzielung von Skalierbarkeit und Verfügbarkeit) auf. In der Internet- und Unternehmensschicht werden VMs mit Lastenausgleich verwendet.

* **Verfügbarkeitsgruppen:** Erstellen Sie eine [Verfügbarkeitsgruppe][azure-availability-sets] für jede Schicht, und stellen Sie mindestens zwei VMs in jeder Schicht bereit.  Dies berechtigt die VMs zu einer höheren [Vereinbarung zum Servicelevel (SLA)][vm-sla] für VMs. Sie können eine einzelne VM in einer Verfügbarkeitsgruppe bereitstellen, die einzelne VM ist jedoch nicht für eine SLA-Garantie qualifiziert, es sei denn, die VM verwendet Azure Premium Storage für alle Betriebssystem- und Datenfestplatten.  
* **Subnetze:** Erstellen Sie für jede Schicht ein separates Subnetz. Geben Sie mit der [CIDR]-Notation den Adressbereich und die Subnetzmaske an. 
* **Lastenausgleichsmodule:** Verwenden Sie ein [Lastenausgleichsmodul mit Internetzugriff][load-balancer-external], um eingehenden Internetdatenverkehr auf die Internetschicht zu verteilen, und ein [internes Lastenausgleichsmodul][load-balancer-internal], um den Netzwerkdatenverkehr von der Internetschicht auf die Unternehmensschicht zu verteilen.
* **Azure DNS:** [Azure DNS][azure-dns] ist ein Hostingdienst für DNS-Domänen, der die Namensauflösung unter Verwendung der Microsoft Azure-Infrastruktur durchführt. Durch das Hosten Ihrer Domänen in Azure können Sie Ihre DNS-Einträge mithilfe der gleichen Anmeldeinformationen, APIs, Tools und Abrechnung wie für die anderen Azure-Dienste verwalten.
* **Jumpbox:** Wird auch als [geschützter Host] bezeichnet. Dies ist eine geschützte VM im Netzwerk, die von Administratoren zum Herstellen der Verbindung mit anderen VMs verwendet wird. Die Jumpbox verfügt über eine NSG, die Remotedatenverkehr nur von öffentlichen IP-Adressen zulässt, die in einer Liste mit sicheren Absendern aufgeführt sind. Die NSG sollte SSH-Datenverkehr (Secure Shell) zulassen.
* **Überwachung:** Mit Überwachungssoftware wie [Nagios], [Zabbix] oder [Icinga] können Informationen über die Antwortzeit, die VM-Betriebszeit und die allgemeine Integrität Ihres Systems bereitstellen. Installieren Sie die Überwachungssoftware auf einer VM, die sich in einem separaten Verwaltungssubnetz befindet.
* <strong>NSGs:</strong> Verwenden Sie [Netzwerksicherheitsgruppen][nsg] (NSGs), um den Netzwerkdatenverkehr im VNET zu beschränken. In der hier gezeigten 3-schichtigen Architektur akzeptiert die Datenbankschicht beispielsweise keinen Datenverkehr vom Web-Front-End, sondern nur von der Unternehmensschicht und dem Verwaltungssubnetz.
* **Apache Cassandra-Datenbank**: Stellt Hochverfügbarkeit in der Datenschicht durch Replikation und Failover bereit.

## <a name="recommendations"></a>Empfehlungen

Ihre Anforderungen können von der hier beschriebenen Architektur abweichen. Verwenden Sie diese Empfehlungen als Startpunkt. 

### <a name="vnet--subnets"></a>VNET/Subnetze

Legen Sie bei der Erstellung des VNET fest, wie viele IP-Adressen Ihre Ressourcen in jedem Subnetz benötigen. Geben Sie mithilfe der [CIDR-Notation] eine Subnetzmaske und einen VNET-Adressbereich an, der für die erforderlichen IP-Adressen groß genug ist. Verwenden Sie einen Adressraum, der in die standardmäßigen [privaten IP-Adressblöcke][private-ip-space] 10.0.0.0/8, 172.16.0.0/12 und 192.168.0.0/16 fällt.

Wählen Sie einen Adressbereich, der sich nicht mit Ihrem lokalen Netzwerk überschneidet, für den Fall, dass Sie später ein Gateway zwischen dem VNET und dem lokalen Netzwerk einrichten müssen. Sobald Sie das VNET erstellt haben, können Sie den Adressbereich nicht mehr ändern.

Entwerfen Sie Subnetze unter Berücksichtigung der Funktionalität und Sicherheitsanforderungen. Alle VMs innerhalb derselben Schicht oder Rolle sollten im selben Subnetz platziert werden, was eine Sicherheitsbegrenzung darstellen kann. Weitere Informationen zum Entwerfen von VNETs und Subnetzen finden Sie unter [Planen und Entwerfen von Azure Virtual Networks][plan-network].

Geben Sie für jedes Subnetz den Adressraum für das Subnetz in CIDR-Notation an. Mit „10.0.0.0/24“ wird beispielsweise ein Bereich von 256 IP-Adressen erstellt. Von diesen sind 251 für VMs nutzbar; die restlichen fünf sind reserviert. Achten Sie darauf, dass sich die Adressbereiche nicht mit anderen Subnetzen überlappen. Weitere Informationen finden Sie unter [Azure Virtual Network – häufig gestellte Fragen][vnet faq].

### <a name="network-security-groups"></a>Netzwerksicherheitsgruppen

Verwenden Sie NSG-Regeln, um den Datenverkehr zwischen den Schichten zu beschränken. In der oben gezeigten 3-schichtigen Architektur kommuniziert die Internetschicht beispielsweise nicht direkt mit der Datenbankschicht. Um dies zu erzwingen, sollte die Datenbankschicht eingehenden Datenverkehr aus dem Subnetz der Internetschicht blockieren.  

1. Erstellen Sie eine NSG, und ordnen Sie diese dem Subnetz der Datenbankschicht zu.
2. Fügen Sie eine Regel hinzu, die den gesamten eingehenden Datenverkehr vom VNET ablehnt. (Verwenden Sie den `VIRTUAL_NETWORK`-Tag in der Regel.) 
3. Fügen Sie eine Regel mit einer höheren Priorität hinzu, die eingehenden Datenverkehr aus dem Subnetz der Unternehmensschicht zulässt. Diese Regel überschreibt die vorherige Regel und ermöglicht die Kommunikation zwischen der Unternehmens- und Datenbankschicht.
4. Fügen Sie eine Regel hinzu, die eingehenden Datenverkehr innerhalb des Subnetzes der Datenbankschicht selbst zulässt. Diese Regel ermöglicht die Kommunikation zwischen VMs in der Datenbankschicht, die für die Replikation und das Failover der Datenbank erforderlich ist.
5. Fügen Sie eine Regel hinzu, die SSH-Datenverkehr vom Subnetz der Jumpbox zulässt. Diese Regel erlaubt Administratoren, über die Jumpbox eine Verbindung mit der Datenbankschicht herzustellen.
   
   > [!NOTE]
   > Eine NSG ist mit [Standardregeln][nsg-rules] versehen, die sämtlichen eingehenden Datenverkehr im VNET zulassen. Diese Regeln können nicht gelöscht, aber durch die Erstellung von Regeln mit höherer Priorität überschrieben werden.
   > 
   > 

### <a name="load-balancers"></a>Load Balancer

Das externe Lastenausgleichsmodul verteilt den Internetdatenverkehr auf die Internetschicht. Erstellen Sie eine öffentliche IP-Adresse für dieses Lastenausgleichsmodul. Weitere Informationen finden Sie unter [Erstellen eines Load Balancers mit Internetzugriff im Azure-Portal][lb-external-create].

Das interne Lastenausgleichsmodul verteilt den Netzwerkdatenverkehr von der Internetschicht auf die Unternehmensschicht. Um diesem Lastenausgleichsmodul eine private IP-Adresse zuzuweisen, erstellen Sie eine Front-End-IP-Konfiguration, und ordnen Sie diese dem Subnetz für die Unternehmensschicht zu. Weitere Informationen finden Sie unter [Erstellen eines internen Lastenausgleichs über das Azure-Portal][lb-internal-create].

### <a name="cassandra"></a>Cassandra

Für die Produktion wird [DataStax Enterprise][datastax] empfohlen, wobei diese Empfehlungen für alle Cassandra-Editionen gelten. Weitere Informationen zur Ausführung von DataStax in Azure finden Sie im [DataStax Enterprise-Bereitstellungshandbuch für Azure][cassandra-in-azure]. 

Fassen Sie die VMs für einen Cassandra-Cluster in einer Verfügbarkeitsgruppe zusammen, um sicherzustellen, dass die Cassandra-Replikate auf verschiedene Fehler- und Upgradedomänen verteilt werden. Weitere Informationen zu Fehler- und Upgradedomänen finden Sie unter [Verwalten der Verfügbarkeit virtueller Computer][azure-availability-sets]. 

Konfigurieren Sie jeweils drei Fehlerdomänen (Maximalwert) und 18 Upgradedomänen pro Verfügbarkeitsgruppe. Dadurch wird die maximale Anzahl von Upgradedomänen erreicht, die nach wie vor gleichmäßig auf die Fehlerdomänen verteilt werden können.   

Konfigurieren Sie Knoten im rackfähigen Modus. Ordnen Sie Fehlerdomänen den Racks in der Datei `cassandra-rackdc.properties` zu.

Es ist kein vor dem Cluster geschaltetes Lastenausgleichsmodul erforderlich. Der Client stellt eine direkte Verbindung mit einem Knoten im Cluster her.

### <a name="jumpbox"></a>Jumpbox

Die Jumpbox setzt minimale Leistungsanforderungen voraus. Wählen Sie daher eine kleine VM-Größe für die Jumpbox wie „Standard A1“. 

Erstellen Sie eine [öffentliche IP-Adresse] für die Jumpbox. Platzieren Sie die Jumpbox in dasselbe VNET wie die anderen VMs, jedoch in ein separates Verwaltungssubnetz.

Verweigern Sie den SSH-Zugriff über das öffentliche Internet auf die VMs, auf denen die Anwendungsworkload ausgeführt wird. Der gesamte SSH-Zugriff auf diese VMs muss stattdessen über die Jumpbox erfolgen. Ein Administrator meldet sich bei der Jumpbox und von der Jumpbox aus dann bei der anderen VM an. Die Jumpbox lässt SSH-Datenverkehr aus dem Internet zu, jedoch nur von bekannten, sicheren IP-Adressen.

Erstellen Sie zum Schutz der Jumpbox eine NSG, und wenden Sie diese auf das Subnetz der Jumpbox an. Fügen Sie eine NSG-Regel hinzu, die ausschließlich SSH-Verbindungen von einer sicheren Gruppe öffentlicher IP-Adressen zulässt. Die NSG kann entweder dem Subnetz oder der Jumpbox-NIC hinzugefügt werden. In diesem Fall empfehlen wir, diese der NIC hinzuzufügen, sodass SSH-Datenverkehr nur an die Jumpbox erlaubt ist, auch wenn Sie andere VMs zum selben Subnetz hinzufügen.

Konfigurieren Sie die NSGs für die anderen Subnetze, um SSH-Datenverkehr aus dem Verwaltungssubnetz zuzulassen.

## <a name="availability-considerations"></a>Überlegungen zur Verfügbarkeit

Platzieren Sie jede Schicht oder VM-Rolle in eine separate Verfügbarkeitsgruppe. 

Durch das Vorhandensein mehrerer VMs in der Datenbankschicht wird nicht automatisch eine hochverfügbare Datenbank erstellt. Bei einer relationalen Datenbank müssen Sie in der Regel Replikation und Failover einsetzen, um Hochverfügbarkeit zu erzielen.  

Wenn Sie eine höhere Verfügbarkeit benötigen, als die [Azure-SLA für VMs][vm-sla] bietet, replizieren Sie die Anwendung über zwei Regionen, und verwenden Sie für das Failover den Azure Traffic Manager. Weitere Informationen finden Sie unter [Ausführen von Linux-VMs in mehreren Regionen für Hochverfügbarkeit][multi-dc].  

## <a name="security-considerations"></a>Sicherheitshinweise

Für die Erstellung einer DMZ zwischen dem öffentlichen Internet und dem virtuellen Azure-Netzwerk sollten Sie eventuell eine virtuelle Netzwerkappliance (Network Virtual Appliance, NVA) hinzufügen. NVA ist ein Oberbegriff für eine virtuelle Appliance, die netzwerkbezogene Aufgaben wie Erstellung von Firewalls, Paketüberprüfung, Überwachung und benutzerdefiniertes Routing ausführen kann. Weitere Informationen finden Sie unter [Implementieren einer DMZ zwischen Azure und dem Internet][dmz].

## <a name="scalability-considerations"></a>Überlegungen zur Skalierbarkeit

Die Lastenausgleichsmodule verteilen den Netzwerkdatenverkehr auf die Internet- und Unternehmensschicht. Führen Sie eine horizontale Skalierung durch, indem Sie neue VM-Instanzen hinzufügen. Beachten Sie, dass die Internet- und Unternehmensschicht je nach Auslastung unabhängig voneinander skaliert werden können. Um das Risiko von Komplikationen zu reduzieren, die durch die Notwendigkeit zur Aufrechterhaltung der Clientaffinität verursacht werden, sollten die VMs in der Internetschicht zustandslos sein. Die VMs, die die Geschäftslogik hosten, sollten ebenfalls zustandslos sein.

## <a name="manageability-considerations"></a>Überlegungen zur Verwaltbarkeit

Vereinfachen Sie die Verwaltung des gesamten Systems durch den Einsatz von zentralisierten Verwaltungstools wie [Azure Automation][azure-administration], [Microsoft Operations Management Suite][operations-management-suite], [Chef][chef] oder [Puppet][puppet]. Diese Tools können Diagnose- und Integritätsinformationen konsolidieren, die von mehreren VMs erfasst wurden, um einen allgemeinen Überblick über das System bereitzustellen.

## <a name="deploy-the-solution"></a>Bereitstellen der Lösung

Eine Bereitstellung für diese Referenzarchitektur ist auf [GitHub][github-folder] verfügbar. 

### <a name="prerequisites"></a>Voraussetzungen

Bevor Sie die Referenzarchitektur in Ihrem eigenen Abonnement bereitstellen können, müssen Sie die folgenden Schritte ausführen.

1. Klonen oder Forken Sie das GitHub-Repository [Referenzarchitekturen][ref-arch-repo], oder laden Sie die entsprechende ZIP-Datei herunter.

2. Vergewissern Sie sich, dass Azure CLI 2.0 auf Ihrem Computer installiert ist. Um die Befehlszeilenschnittstelle zu installieren, befolgen Sie die Anweisungen unter [Installieren von Azure CLI 2.0][azure-cli-2].

3. Installieren Sie das npm-Paket mit den [Azure Bausteinen][azbb].

   ```bash
   npm install -g @mspnp/azure-building-blocks
   ```

4. Melden Sie sich über eine Eingabeaufforderung, eine bash-Eingabeaufforderung oder die PowerShell-Eingabeaufforderung bei Ihrem Azure-Konto an. Verwenden Sie dazu die unten aufgeführten Befehle, und befolgen Sie die Anweisungen.

   ```bash
   az login
   ```

### <a name="deploy-the-solution-using-azbb"></a>Bereitstellen der Lösung mit azbb

Um die Linux VMs für eine n-schichtige Anwendungsreferenzarchitektur bereitzustellen, führen Sie die folgenden Schritte aus:

1. Navigieren Sie zum `virtual-machines\n-tier-linux`-Ordner für das Repository, das Sie oben in Schritt 1 der Voraussetzungen als Klon erstellt haben.

2. Die Parameterdatei gibt einen Standard-Administratorbenutzernamen und ein Standardkennwort für jede VM in der Bereitstellung an. Sie müssen diese vor der Bereitstellung der Referenzarchitektur ändern. Öffnen Sie die `n-tier-linux.json`-Datei, und ersetzen Sie jedes Feld **adminUsername** und **adminPassword** durch neue Einstellungen.   Speichern Sie die Datei .

3. Stellen Sie die Referenzarchitektur mithilfe des Befehlszeilentools **azbb** bereit, wie unten dargestellt.

   ```bash
   azbb -s <your subscription_id> -g <your resource_group_name> -l <azure region> -p n-tier-linux.json --deploy
   ```

Weitere Informationen zum Bereitstellen dieser Beispielreferenzarchitektur mithilfe von Azure-Bausteinen finden Sie im [GitHub-Repository][git].

<!-- links -->
[multi-dc]: multi-region-application.md
[dmz]: ../dmz/secure-vnet-dmz.md
[multi-vm]: ./multi-vm.md
[naming conventions]: /azure/guidance/guidance-naming-conventions
[azbb]: https://github.com/mspnp/template-building-blocks/wiki/Install-Azure-Building-Blocks
[azure-administration]: /azure/automation/automation-intro
[azure-availability-sets]: /azure/virtual-machines/virtual-machines-linux-manage-availability
[azure-cli-2]: https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest
[azure-dns]: /azure/dns/dns-overview
[geschützter Host]: https://en.wikipedia.org/wiki/Bastion_host
[cassandra-in-azure]: https://docs.datastax.com/en/datastax_enterprise/4.5/datastax_enterprise/install/installAzure.html
[CIDR]: https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing
[chef]: https://www.chef.io/solutions/azure/
[datastax]: http://www.datastax.com/products/datastax-enterprise
[git]: https://github.com/mspnp/template-building-blocks
[github-folder]: https://github.com/mspnp/reference-architectures/tree/master/virtual-machines/n-tier-linux
[lb-external-create]: /azure/load-balancer/load-balancer-get-started-internet-portal
[lb-internal-create]: /azure/load-balancer/load-balancer-get-started-ilb-arm-portal
[load-balancer-external]: /azure/load-balancer/load-balancer-internet-overview
[load-balancer-internal]: /azure/load-balancer/load-balancer-internal-overview
[nsg]: /azure/virtual-network/virtual-networks-nsg
[nsg-rules]: /azure/azure-resource-manager/best-practices-resource-manager-security#network-security-groups
[operations-management-suite]: https://www.microsoft.com/server-cloud/operations-management-suite/overview.aspx
[plan-network]: /azure/virtual-network/virtual-network-vnet-plan-design-arm
[private-ip-space]: https://en.wikipedia.org/wiki/Private_network#Private_IPv4_address_spaces
[öffentliche IP-Adresse]: /azure/virtual-network/virtual-network-ip-addresses-overview-arm
[puppet]: https://puppetlabs.com/blog/managing-azure-virtual-machines-puppet
[ref-arch-repo]: https://github.com/mspnp/reference-architectures
[vm-sla]: https://azure.microsoft.com/support/legal/sla/virtual-machines
[vnet faq]: /azure/virtual-network/virtual-networks-faq
[visio-download]: https://archcenter.blob.core.windows.net/cdn/vm-reference-architectures.vsdx
[Nagios]: https://www.nagios.org/
[Zabbix]: http://www.zabbix.com/
[Icinga]: http://www.icinga.org/
[0]: ./images/n-tier-diagram.png "n-schichtige Architektur mit Microsoft Azure"

