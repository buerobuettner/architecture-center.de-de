---
title: Kompensierende Transaktion
description: "Machen Sie durch eine Reihe von Schritten ausgeführte Arbeit rückgängig, die zusammen einen letztlich konsistenten Vorgang definieren."
keywords: Entwurfsmuster
author: dragon119
ms.date: 06/23/2017
pnp.series.title: Cloud Design Patterns
pnp.pattern.categories: resiliency
ms.openlocfilehash: a822de990d6ce933024207073b110e98f8da40bf
ms.sourcegitcommit: 8ab30776e0c4cdc16ca0dcc881960e3108ad3e94
ms.translationtype: HT
ms.contentlocale: de-DE
ms.lasthandoff: 12/08/2017
---
# <a name="compensating-transaction-pattern"></a><span data-ttu-id="94942-104">Muster „Kompensierende Transaktion“</span><span class="sxs-lookup"><span data-stu-id="94942-104">Compensating Transaction pattern</span></span>

[!INCLUDE [header](../_includes/header.md)]

<span data-ttu-id="94942-105">Machen Sie die Vorgänge rückgängig, die für eine Reihe von Schritten ausgeführt wurden (die zusammen schlussendlich einen letztlich konsistenten Vorgang bilden), wenn mindestens einer dieser Schritte fehlerhaft ist.</span><span class="sxs-lookup"><span data-stu-id="94942-105">Undo the work performed by a series of steps, which together define an eventually consistent operation, if one or more of the steps fail.</span></span> <span data-ttu-id="94942-106">Vorgänge, die dem Modell der letztlichen Konsistenz folgen, sind häufig in cloudbasierten Anwendungen anzutreffen, die komplexe Geschäftsprozesse und -workflows implementieren.</span><span class="sxs-lookup"><span data-stu-id="94942-106">Operations that follow the eventual consistency model are commonly found in cloud-hosted applications that implement complex business processes and workflows.</span></span>

## <a name="context-and-problem"></a><span data-ttu-id="94942-107">Kontext und Problem</span><span class="sxs-lookup"><span data-stu-id="94942-107">Context and problem</span></span>

<span data-ttu-id="94942-108">Anwendungen, die häufig in der Cloud ausgeführt werden, ändern Daten.</span><span class="sxs-lookup"><span data-stu-id="94942-108">Applications running in the cloud frequently modify data.</span></span> <span data-ttu-id="94942-109">Diese Daten können auf verschiedene Datenquellen verteilt sein, die sich an unterschiedlichen geographischen Standorten befinden.</span><span class="sxs-lookup"><span data-stu-id="94942-109">This data might be spread across various data sources held in different geographic locations.</span></span> <span data-ttu-id="94942-110">Um in einer verteilten Umgebung Konflikte zu vermeiden und die Leistung zu verbessern, sollte eine Anwendung nicht dafür eingesetzt werden, eine hohe Transaktionskonsistenz zu gewährleisten.</span><span class="sxs-lookup"><span data-stu-id="94942-110">To avoid contention and improve performance in a distributed environment, an application shouldn't try to provide strong transactional consistency.</span></span> <span data-ttu-id="94942-111">Vielmehr sollte die Anwendung letztliche Konsistenz implementieren.</span><span class="sxs-lookup"><span data-stu-id="94942-111">Rather, the application should implement eventual consistency.</span></span> <span data-ttu-id="94942-112">In diesem Modell besteht ein typischer Geschäftsvorgang aus einer Reihe von separaten Schritten.</span><span class="sxs-lookup"><span data-stu-id="94942-112">In this model, a typical business operation consists of a series of separate steps.</span></span> <span data-ttu-id="94942-113">Während der Ausführung dieser Schritte kann die Gesamtsicht auf den Systemzustand inkonsistent sein, doch wenn der Vorgang abgeschlossen ist und alle Schritte ausgeführt wurden, sollte das System wieder konsistent sein.</span><span class="sxs-lookup"><span data-stu-id="94942-113">While these steps are being performed, the overall view of the system state might be inconsistent, but when the operation has completed and all of the steps have been executed the system should become consistent again.</span></span>

> <span data-ttu-id="94942-114">Der [Datenkonsistenzprimer](https://msdn.microsoft.com/library/dn589800.aspx) liefert Informationen darüber, warum verteilte Transaktionen nicht gut skaliert werden können, sowie über die Prinzipien des Modells der letztlichen Konsistenz.</span><span class="sxs-lookup"><span data-stu-id="94942-114">The [Data Consistency Primer](https://msdn.microsoft.com/library/dn589800.aspx) provides information about why distributed transactions don't scale well, and the principles of the eventual consistency model.</span></span>

<span data-ttu-id="94942-115">Eine Herausforderung beim Modell der letztlichen Konsistenz stellt die Frage dar, wie mit einem fehlerhaften Schritt umgegangen wird.</span><span class="sxs-lookup"><span data-stu-id="94942-115">A challenge in the eventual consistency model is how to handle a step that has failed.</span></span> <span data-ttu-id="94942-116">In diesem Fall kann es notwendig sein, alle Arbeiten rückgängig zu machen, die durch die vorhergehenden Schritte des Vorgangs ausgeführt wurden.</span><span class="sxs-lookup"><span data-stu-id="94942-116">In this case it might be necessary to undo all of the work completed by the previous steps in the operation.</span></span> <span data-ttu-id="94942-117">Für die Daten kann jedoch nicht ohne Weiteres ein Rollback ausgeführt werden, da andere gleichzeitige Instanzen der Anwendung diese möglicherweise geändert haben.</span><span class="sxs-lookup"><span data-stu-id="94942-117">However, the data can't simply be rolled back because other concurrent instances of the application might have changed it.</span></span> <span data-ttu-id="94942-118">Selbst in Fällen, in denen die Daten nicht durch eine gleichzeitige Instanz verändert wurden, kann das Rückgängigmachen eines Schrittes nicht nur eine Frage der Wiederherstellung des ursprünglichen Zustandes sein.</span><span class="sxs-lookup"><span data-stu-id="94942-118">Even in cases where the data hasn't been changed by a concurrent instance, undoing a step might not simply be a matter of restoring the original state.</span></span> <span data-ttu-id="94942-119">So kann es notwendig sein, verschiedene geschäftsspezifische Regeln anzuwenden (siehe die im Abschnitt „Beispiel“ beschriebene Reisewebsite).</span><span class="sxs-lookup"><span data-stu-id="94942-119">It might be necessary to apply various business-specific rules (see the travel website described in the Example section).</span></span>

<span data-ttu-id="94942-120">Wenn sich ein Vorgang, der letztliche Konsistenz implementiert, über mehrere heterogene Datenspeicher erstreckt, muss zum Rückgängigmachen der Schritte des Vorgangs nacheinander jeder einzelne Datenspeicher aufgerufen werden.</span><span class="sxs-lookup"><span data-stu-id="94942-120">If an operation that implements eventual consistency spans several heterogeneous data stores, undoing the steps in the operation will require visiting each data store in turn.</span></span> <span data-ttu-id="94942-121">Alle in den einzelnen Datenspeichern durchgeführte Aufgaben müssen zuverlässig rückgängig gemacht werden, damit das System nicht in einem inkonsistenten Zustand verbleibt.</span><span class="sxs-lookup"><span data-stu-id="94942-121">The work performed in every data store must be undone reliably to prevent the system from remaining inconsistent.</span></span>

<span data-ttu-id="94942-122">Nicht alle Daten, die von einem Vorgang, der letztliche Konsistenz implementiert, betroffen sind, können in einer Datenbank gespeichert werden.</span><span class="sxs-lookup"><span data-stu-id="94942-122">Not all data affected by an operation that implements eventual consistency might be held in a database.</span></span> <span data-ttu-id="94942-123">In einer Umgebung mit serviceorientierter Architektur (SOA) könnte ein Vorgang eine Aktion in einem Dienst aufrufen und eine Änderung des Zustands dieses Diensts bewirken.</span><span class="sxs-lookup"><span data-stu-id="94942-123">In a service oriented architecture (SOA) environment an operation could invoke an action in a service, and cause a change in the state held by that service.</span></span> <span data-ttu-id="94942-124">Um den Vorgang rückgängig zu machen, muss auch diese Zustandsänderung rückgängig gemacht werden.</span><span class="sxs-lookup"><span data-stu-id="94942-124">To undo the operation, this state change must also be undone.</span></span> <span data-ttu-id="94942-125">Hierfür muss der Dienst eventuell erneut aufgerufen und eine weitere Aktion ausgeführt werden, die die Auswirkungen der ersten Aktion rückgängig macht.</span><span class="sxs-lookup"><span data-stu-id="94942-125">This can involve invoking the service again and performing another action that reverses the effects of the first.</span></span>

## <a name="solution"></a><span data-ttu-id="94942-126">Lösung</span><span class="sxs-lookup"><span data-stu-id="94942-126">Solution</span></span>

<span data-ttu-id="94942-127">Die Lösung besteht darin, eine kompensierende Transaktion zu implementieren.</span><span class="sxs-lookup"><span data-stu-id="94942-127">The solution is to implement a compensating transaction.</span></span> <span data-ttu-id="94942-128">Die Schritte einer kompensierenden Transaktion müssen die Auswirkungen der Schritte des ursprünglichen Vorgangs rückgängig machen.</span><span class="sxs-lookup"><span data-stu-id="94942-128">The steps in a compensating transaction must undo the effects of the steps in the original operation.</span></span> <span data-ttu-id="94942-129">Eine kompensierende Transaktion kann möglicherweise nicht einfach den aktuellen Zustand durch den Zustand ersetzen, in dem sich das System zu Beginn des Vorgangs befand, da durch diese Vorgehensweise Änderungen überschrieben werden könnten, die von anderen gleichzeitigen Instanzen einer Anwendung vorgenommen wurden.</span><span class="sxs-lookup"><span data-stu-id="94942-129">A compensating transaction might not be able to simply replace the current state with the state the system was in at the start of the operation because this approach could overwrite changes made by other concurrent instances of an application.</span></span> <span data-ttu-id="94942-130">Stattdessen muss es sich um einen intelligenten Prozess handeln, der alle von gleichzeitigen Instanzen durchgeführten Aufgaben berücksichtigt.</span><span class="sxs-lookup"><span data-stu-id="94942-130">Instead, it must be an intelligent process that takes into account any work done by concurrent instances.</span></span> <span data-ttu-id="94942-131">Dieser Prozess ist in der Regel anwendungsspezifisch und hängt von der Art der Aufgaben ab, die im ursprünglichen Vorgang durchgeführt wurden.</span><span class="sxs-lookup"><span data-stu-id="94942-131">This process will usually be application specific, driven by the nature of the work performed by the original operation.</span></span>

<span data-ttu-id="94942-132">Eine übliche Vorgehensweise besteht darin, einen Workflow einzusetzen, um einen letztlich konsistenten Vorgang zu implementieren, der eine Kompensierung erfordert.</span><span class="sxs-lookup"><span data-stu-id="94942-132">A common approach is to use a workflow to implement an eventually consistent operation that requires compensation.</span></span> <span data-ttu-id="94942-133">Im weiteren Verlauf des ursprünglichen Vorgangs zeichnet das System Informationen über jeden einzelnen Schritt auf und zeigt an, wie die in diesem Schritt durchgeführten Aufgaben rückgängig gemacht werden können.</span><span class="sxs-lookup"><span data-stu-id="94942-133">As the original operation proceeds, the system records information about each step and how the work performed by that step can be undone.</span></span> <span data-ttu-id="94942-134">Wenn beim Vorgang an irgendeiner Stelle ein Fehler auftritt, geht der Workflow die abgeschlossenen Schritte rückwärts durch und führt die Aufgabe zum Rückgängigmachen jedes Schritts durch.</span><span class="sxs-lookup"><span data-stu-id="94942-134">If the operation fails at any point, the workflow rewinds back through the steps it's completed and performs the work that reverses each step.</span></span> <span data-ttu-id="94942-135">Beachten Sie, dass eine kompensierende Transaktion die Aufgabe möglicherweise nicht in genau der umgekehrten Reihenfolge des ursprünglichen Vorgangs rückgängig machen muss, und dass es möglich ist, einige der Schritte zum Rückgängigmachen parallel auszuführen.</span><span class="sxs-lookup"><span data-stu-id="94942-135">Note that a compensating transaction might not have to undo the work in the exact reverse order of the original operation, and it might be possible to perform some of the undo steps in parallel.</span></span>

> <span data-ttu-id="94942-136">Diese Vorgehensweise ähnelt der Sagas-Strategie, die im [Blog von Clemens Vasters](http://vasters.com/clemensv/2012/09/01/Sagas.aspx) diskutiert wird.</span><span class="sxs-lookup"><span data-stu-id="94942-136">This approach is similar to the Sagas strategy discussed in [Clemens Vasters’ blog](http://vasters.com/clemensv/2012/09/01/Sagas.aspx).</span></span>

<span data-ttu-id="94942-137">Eine kompensierende Transaktion ist auch ein letztlich konsistenter Vorgang und kann auch Fehler verursachen.</span><span class="sxs-lookup"><span data-stu-id="94942-137">A compensating transaction is also an eventually consistent operation and it could also fail.</span></span> <span data-ttu-id="94942-138">Das System sollte in der Lage sein, die kompensierende Transaktion an der Stelle, an der der Fehler aufgetreten ist, wieder aufzunehmen und fortzusetzen.</span><span class="sxs-lookup"><span data-stu-id="94942-138">The system should be able to resume the compensating transaction at the point of failure and continue.</span></span> <span data-ttu-id="94942-139">Da ein fehlerhafter Schritt eventuell wiederholt werden muss, sollten die Schritte in einer kompensierenden Transaktion als idempotente Befehle definiert werden.</span><span class="sxs-lookup"><span data-stu-id="94942-139">It might be necessary to repeat a step that's failed, so the steps in a compensating transaction should be defined as idempotent commands.</span></span> <span data-ttu-id="94942-140">Weitere Informationen finden Sie unter [Idempotenzmuster](http://blog.jonathanoliver.com/idempotency-patterns/) im Blog von Jonathan Oliver.</span><span class="sxs-lookup"><span data-stu-id="94942-140">For more information, see [Idempotency Patterns](http://blog.jonathanoliver.com/idempotency-patterns/) on Jonathan Oliver’s blog.</span></span>

<span data-ttu-id="94942-141">In einigen Fällen kann eine Wiederherstellung nach einem fehlerhaften Schritt nur durch manuelles Eingreifen durchgeführt werden.</span><span class="sxs-lookup"><span data-stu-id="94942-141">In some cases it might not be possible to recover from a step that has failed except through manual intervention.</span></span> <span data-ttu-id="94942-142">In diesen Situationen sollte das System eine Warnung auslösen und so viele Informationen wie möglich über die Ursache des Fehlers bereitstellen.</span><span class="sxs-lookup"><span data-stu-id="94942-142">In these situations the system should raise an alert and provide as much information as possible about the reason for the failure.</span></span>

## <a name="issues-and-considerations"></a><span data-ttu-id="94942-143">Probleme und Überlegungen</span><span class="sxs-lookup"><span data-stu-id="94942-143">Issues and considerations</span></span>

<span data-ttu-id="94942-144">Beachten Sie die folgenden Punkte bei der Entscheidung, wie dieses Muster implementiert werden soll:</span><span class="sxs-lookup"><span data-stu-id="94942-144">Consider the following points when deciding how to implement this pattern:</span></span>

<span data-ttu-id="94942-145">Es mag nicht einfach sein, festzustellen, wann bei einem Schritt in einem Vorgang, der letztliche Konsistenz implementiert, ein Fehler aufgetreten ist.</span><span class="sxs-lookup"><span data-stu-id="94942-145">It might not be easy to determine when a step in an operation that implements eventual consistency has failed.</span></span> <span data-ttu-id="94942-146">Bei einem Schritt muss nicht sofort ein Fehler auftreten, er könnte auch blockiert werden.</span><span class="sxs-lookup"><span data-stu-id="94942-146">A step might not fail immediately, but instead could block.</span></span> <span data-ttu-id="94942-147">Eventuell muss eine Art von Timeoutmechanismus implementiert werden.</span><span class="sxs-lookup"><span data-stu-id="94942-147">It might be necessary to implement some form of time-out mechanism.</span></span>

<span data-ttu-id="94942-148">Eine Kompensationslogik lässt sich nicht ohne Weiteres verallgemeinern.</span><span class="sxs-lookup"><span data-stu-id="94942-148">-Compensation logic isn't easily generalized.</span></span> <span data-ttu-id="94942-149">Eine kompensierende Transaktion ist anwendungsspezifisch.</span><span class="sxs-lookup"><span data-stu-id="94942-149">A compensating transaction is application specific.</span></span> <span data-ttu-id="94942-150">Sie setzt voraus, dass die Anwendung über ausreichende Informationen verfügt, um die Auswirkungen jedes einzelnen Schrittes in einem fehlerhaften Vorgang rückgängig machen zu können.</span><span class="sxs-lookup"><span data-stu-id="94942-150">It relies on the application having sufficient information to be able to undo the effects of each step in a failed operation.</span></span>

<span data-ttu-id="94942-151">Sie sollten die Schritte einer kompensierenden Transaktion als idempotente Befehle definieren.</span><span class="sxs-lookup"><span data-stu-id="94942-151">You should define the steps in a compensating transaction as idempotent commands.</span></span> <span data-ttu-id="94942-152">Dadurch können die Schritte wiederholt werden, wenn bei der kompensierenden Transaktion selbst ein Fehler auftritt.</span><span class="sxs-lookup"><span data-stu-id="94942-152">This enables the steps to be repeated if the compensating transaction itself fails.</span></span>

<span data-ttu-id="94942-153">Die Infrastruktur, die die Schritte des ursprünglichen Vorgangs und die kompensierende Transaktion abwickelt, muss stabil sein.</span><span class="sxs-lookup"><span data-stu-id="94942-153">The infrastructure that handles the steps in the original operation, and the compensating transaction, must be resilient.</span></span> <span data-ttu-id="94942-154">Sie darf nicht die Informationen verlieren, die zum Kompensieren eines fehlerhaften Schrittes erforderlich sind, und muss in der Lage sein, den Fortschritt der Kompensationslogik zuverlässig zu überwachen.</span><span class="sxs-lookup"><span data-stu-id="94942-154">It must not lose the information required to compensate for a failing step, and it must be able to reliably monitor the progress of the compensation logic.</span></span>

<span data-ttu-id="94942-155">Eine kompensierende Transaktion gibt die Daten im System nicht unbedingt in dem Zustand zurück, in dem sich diese zu Beginn des ursprünglichen Vorgangs befanden.</span><span class="sxs-lookup"><span data-stu-id="94942-155">A compensating transaction doesn't necessarily return the data in the system to the state it was in at the start of the original operation.</span></span> <span data-ttu-id="94942-156">Stattdessen kompensiert sie die Aufgabe, die durch die erfolgreich abgeschlossenen Schritte vor dem Fehlervorkommnis im Vorgang durchgeführt wurde.</span><span class="sxs-lookup"><span data-stu-id="94942-156">Instead, it compensates for the work performed by the steps that completed successfully before the operation failed.</span></span>

<span data-ttu-id="94942-157">Die Reihenfolge der Schritte in der kompensierenden Transaktion muss nicht unbedingt in der genau gegenteiligen Reihenfolge der Schritte im ursprünglichen Vorgang erfolgen.</span><span class="sxs-lookup"><span data-stu-id="94942-157">The order of the steps in the compensating transaction doesn't necessarily have to be the exact opposite of the steps in the original operation.</span></span> <span data-ttu-id="94942-158">Beispielsweise könnte ein Datenspeicher empfindlicher auf Inkonsistenzen reagieren als ein anderer Datenspeicher. Daher sollten die Schritte in der kompensierenden Transaktion, die die Änderungen an diesem Speicher rückgängig machen, zuerst durchgeführt werden.</span><span class="sxs-lookup"><span data-stu-id="94942-158">For example, one data store might be more sensitive to inconsistencies than another, and so the steps in the compensating transaction that undo the changes to this store should occur first.</span></span>

<span data-ttu-id="94942-159">Das Festlegen einer kurzfristigen, auf einem Timeout basierende Sperre für jede Ressource, die zur Durchführung eines Vorgangs erforderlich ist, und das vorzeitige Abrufen dieser Ressourcen kann die Wahrscheinlichkeit erhöhen, dass die Gesamtaktivität erfolgreich durchgeführt wird.</span><span class="sxs-lookup"><span data-stu-id="94942-159">Placing a short-term timeout-based lock on each resource that's required to complete an operation, and obtaining these resources in advance, can help increase the likelihood that the overall activity will succeed.</span></span> <span data-ttu-id="94942-160">Die Aufgabe sollte erst dann durchgeführt werden, wenn alle Ressourcen bezogen wurden.</span><span class="sxs-lookup"><span data-stu-id="94942-160">The work should be performed only after all the resources have been acquired.</span></span> <span data-ttu-id="94942-161">Alle Aktionen müssen abgeschlossen sein, bevor die Sperren ablaufen.</span><span class="sxs-lookup"><span data-stu-id="94942-161">All actions must be finalized before the locks expire.</span></span>

<span data-ttu-id="94942-162">Ziehen Sie die Verwendung einer Wiederholungslogik in Betracht, die mehr Fehler toleriert als üblich, um Fehler zu minimieren, die eine kompensierende Transaktion auslösen.</span><span class="sxs-lookup"><span data-stu-id="94942-162">Consider using retry logic that is more forgiving than usual to minimize failures that trigger a compensating transaction.</span></span> <span data-ttu-id="94942-163">Wenn bei einem Schritt eines Vorgangs zur Implementierung von letztlicher Konsistenz ein Fehler auftritt, behandeln Sie den Fehler als vorübergehende Ausnahme, und wiederholen Sie den Schritt.</span><span class="sxs-lookup"><span data-stu-id="94942-163">If a step in an operation that implements eventual consistency fails, try handling the failure as a transient exception and repeat the step.</span></span> <span data-ttu-id="94942-164">Beenden Sie den Vorgang, und initiieren Sie eine kompensierende Transaktion, wenn bei einem Schritt wiederholt und unwiederbringlich ein Fehler auftritt.</span><span class="sxs-lookup"><span data-stu-id="94942-164">Only stop the operation and initiate a compensating transaction if a step fails repeatedly or irrecoverably.</span></span>

> <span data-ttu-id="94942-165">Viele der Herausforderungen bei der Implementierung einer kompensierenden Transaktion sind mit denen bei der Implementierung von letztlicher Konsistenz identisch.</span><span class="sxs-lookup"><span data-stu-id="94942-165">Many of the challenges of implementing a compensating transaction are the same as those with implementing eventual consistency.</span></span> <span data-ttu-id="94942-166">Weitere Informationen hierzu finden Sie im [Datenkonsistenzprimer](https://msdn.microsoft.com/library/dn589800.aspx) im Abschnitt „Überlegungen zur Implementierung von letztlicher Konsistenz“.</span><span class="sxs-lookup"><span data-stu-id="94942-166">See the section Considerations for Implementing Eventual Consistency in the [Data Consistency Primer](https://msdn.microsoft.com/library/dn589800.aspx) for more information.</span></span>

## <a name="when-to-use-this-pattern"></a><span data-ttu-id="94942-167">Verwendung dieses Musters</span><span class="sxs-lookup"><span data-stu-id="94942-167">When to use this pattern</span></span>

<span data-ttu-id="94942-168">Verwenden Sie dieses Muster nur bei Vorgängen, die bei auftretenden Fehlern rückgängig gemacht werden müssen.</span><span class="sxs-lookup"><span data-stu-id="94942-168">Use this pattern only for operations that must be undone if they fail.</span></span> <span data-ttu-id="94942-169">Entwerfen Sie möglichst Lösungen, mit denen Sie die Komplexität kompensierender Transaktionen vermeiden können.</span><span class="sxs-lookup"><span data-stu-id="94942-169">If possible, design solutions to avoid the complexity of requiring compensating transactions.</span></span>

## <a name="example"></a><span data-ttu-id="94942-170">Beispiel</span><span class="sxs-lookup"><span data-stu-id="94942-170">Example</span></span>

<span data-ttu-id="94942-171">Über eine Reisewebsite können Kunden Reiserouten buchen.</span><span class="sxs-lookup"><span data-stu-id="94942-171">A travel website lets customers book itineraries.</span></span> <span data-ttu-id="94942-172">Eine einzelne Reiseroute kann aus einer Reihe von Flügen und Hotels bestehen.</span><span class="sxs-lookup"><span data-stu-id="94942-172">A single itinerary might comprise a series of flights and hotels.</span></span> <span data-ttu-id="94942-173">Ein Kunde, der von Seattle nach London und dann weiter nach Paris reist, könnte die bei der Erstellung einer Reiseroute folgenden Schritte durchführen:</span><span class="sxs-lookup"><span data-stu-id="94942-173">A customer traveling from Seattle to London and then on to Paris could perform the following steps when creating an itinerary:</span></span>

1. <span data-ttu-id="94942-174">Buchen Sie einen Sitzplatz auf dem Flug F1 von Seattle nach London.</span><span class="sxs-lookup"><span data-stu-id="94942-174">Book a seat on flight F1 from Seattle to London.</span></span>
2. <span data-ttu-id="94942-175">Buchen Sie einen Sitzplatz auf dem Flug F2 von London nach Paris.</span><span class="sxs-lookup"><span data-stu-id="94942-175">Book a seat on flight F2 from London to Paris.</span></span>
3. <span data-ttu-id="94942-176">Buchen Sie einen Sitzplatz auf dem Flug F3 von Paris nach Seattle.</span><span class="sxs-lookup"><span data-stu-id="94942-176">Book a seat on flight F3 from Paris to Seattle.</span></span>
4. <span data-ttu-id="94942-177">Reservieren Sie ein Zimmer im Hotel H1 in London.</span><span class="sxs-lookup"><span data-stu-id="94942-177">Reserve a room at hotel H1 in London.</span></span>
5. <span data-ttu-id="94942-178">Reservieren Sie ein Zimmer im Hotel H2 in Paris.</span><span class="sxs-lookup"><span data-stu-id="94942-178">Reserve a room at hotel H2 in Paris.</span></span>

<span data-ttu-id="94942-179">Diese Schritte bilden einen letztlich konsistenten Vorgang, obwohl jeder Schritt eine separate Aktion darstellt.</span><span class="sxs-lookup"><span data-stu-id="94942-179">These steps constitute an eventually consistent operation, although each step is a separate action.</span></span> <span data-ttu-id="94942-180">Daher muss das System nicht nur diese Schritte durchführen, sondern auch die entgegensetzten Vorgänge erfassen, die zum Rückgängigmachen jedes Schritts notwendig sind, falls der Kunde beschließt, die Reiseroute zu stornieren.</span><span class="sxs-lookup"><span data-stu-id="94942-180">Therefore, as well as performing these steps, the system must also record the counter operations necessary to undo each step in case the customer decides to cancel the itinerary.</span></span> <span data-ttu-id="94942-181">Die zur Durchführung der entgegengesetzten Vorgänge erforderlichen Schritte können dann als kompensierende Transaktion ausgeführt werden.</span><span class="sxs-lookup"><span data-stu-id="94942-181">The steps necessary to perform the counter operations can then run as a compensating transaction.</span></span>

<span data-ttu-id="94942-182">Beachten Sie, dass die Schritte in der kompensierenden Transaktion möglicherweise nicht das genaue Gegenteil der ursprünglichen Schritte darstellen und die Logik in jedem Schritt in der kompensierenden Transaktion allen geschäftsspezifischen Regeln entsprechen muss.</span><span class="sxs-lookup"><span data-stu-id="94942-182">Notice that the steps in the compensating transaction might not be the exact opposite of the original steps, and the logic in each step in the compensating transaction must take into account any business-specific rules.</span></span> <span data-ttu-id="94942-183">Beispielsweise ist ein Kunden bei Stornierung eines Sitzplatzes auf einem Flug möglicherweise nicht zur vollständigen Rückerstattung des gezahlten Betrages berechtigt.</span><span class="sxs-lookup"><span data-stu-id="94942-183">For example, unbooking a seat on a flight might not entitle the customer to a complete refund of any money paid.</span></span> <span data-ttu-id="94942-184">Die Abbildung zeigt die Generierung einer kompensierenden Transaktion, um eine zeitintensive Transaktion zur Buchung einer Reiseroute rückgängig zu machen.</span><span class="sxs-lookup"><span data-stu-id="94942-184">The figure illustrates generating a compensating transaction to undo a long-running transaction to book a travel itinerary.</span></span>

![Generieren einer kompensierenden Transaktion zum Rückgängigmachen von zeitintensiven Transaktionen für die Buchung einer Reiseroute](./_images/compensating-transaction-diagram.png)


> <span data-ttu-id="94942-186">Abhängig vom Entwurf der Kompensationslogik für die einzelnen Schritte können die Schritte der kompensierenden Transaktion parallel ausgeführt werden.</span><span class="sxs-lookup"><span data-stu-id="94942-186">It might be possible for the steps in the compensating transaction to be performed in parallel, depending on how you've designed the compensating logic for each step.</span></span>

<span data-ttu-id="94942-187">In vielen Unternehmenslösungen ist beim Auftreten eines Fehlers bei einem einzelnen Schritt nicht immer ein Rollback des Systems mit einer kompensierenden Transaktion erforderlich.</span><span class="sxs-lookup"><span data-stu-id="94942-187">In many business solutions, failure of a single step doesn't always necessitate rolling the system back by using a compensating transaction.</span></span> <span data-ttu-id="94942-188">Wenn der Kunde nicht in der Lage ist, – beispielsweise nach der Buchung der Flüge F1, F2 und F3 im Szenario der Reisewebsite – ein Zimmer im Hotel H1 zu reservieren, ist es besser, dem Kunden ein Zimmer in einem anderen Hotel in derselben Stadt anzubieten, als die Flüge zu stornieren.</span><span class="sxs-lookup"><span data-stu-id="94942-188">For example, if&mdash;after having booked flights F1, F2, and F3 in the travel website scenario&mdash;the customer is unable to reserve a room at hotel H1, it's preferable to offer the customer a room at a different hotel in the same city rather than canceling the flights.</span></span> <span data-ttu-id="94942-189">Der Kunde hat nach wie vor die Möglichkeit, eine Stornierung durchzuführen (in diesem Fall wird die kompensierende Transaktion ausgeführt, und die Buchungen für die Flüge F1, F2 und F3 werden rückgängig gemacht), aber diese Entscheidung sollte vom Kunden und nicht vom System getroffen werden.</span><span class="sxs-lookup"><span data-stu-id="94942-189">The customer can still decide to cancel (in which case the compensating transaction runs and undoes the bookings made on flights F1, F2, and F3), but this decision should be made by the customer rather than by the system.</span></span>

## <a name="related-patterns-and-guidance"></a><span data-ttu-id="94942-190">Zugehörige Muster und Anleitungen</span><span class="sxs-lookup"><span data-stu-id="94942-190">Related patterns and guidance</span></span>

<span data-ttu-id="94942-191">Die folgenden Muster und Anweisungen können für die Implementierung dieses Musters ebenfalls relevant sein:</span><span class="sxs-lookup"><span data-stu-id="94942-191">The following patterns and guidance might also be relevant when implementing this pattern:</span></span>

- <span data-ttu-id="94942-192">[Datenkonsistenzprimer](https://msdn.microsoft.com/library/dn589800.aspx):</span><span class="sxs-lookup"><span data-stu-id="94942-192">[Data Consistency Primer](https://msdn.microsoft.com/library/dn589800.aspx).</span></span> <span data-ttu-id="94942-193">Das Muster „Kompensierende Transaktion“ wird häufig verwendet, um Vorgänge rückgängig zu machen, die das Modell der letztlichen Konsistenz implementieren.</span><span class="sxs-lookup"><span data-stu-id="94942-193">The Compensating Transaction pattern is often used to undo operations that implement the eventual consistency model.</span></span> <span data-ttu-id="94942-194">Dieser Primer liefert Informationen über die Vor- und Nachteile von letztlicher Konsistenz.</span><span class="sxs-lookup"><span data-stu-id="94942-194">This primer provides information on the benefits and tradeoffs of eventual consistency.</span></span>

- <span data-ttu-id="94942-195">[Muster „Scheduler-Agent-Supervisor“](scheduler-agent-supervisor.md):</span><span class="sxs-lookup"><span data-stu-id="94942-195">[Scheduler-Agent-Supervisor Pattern](scheduler-agent-supervisor.md).</span></span> <span data-ttu-id="94942-196">Beschreibt die Implementierung stabiler Systeme, die Geschäftsvorgänge durchführen, bei denen verteilte Dienste und Ressourcen zum Einsatz kommen.</span><span class="sxs-lookup"><span data-stu-id="94942-196">Describes how to implement resilient systems that perform business operations that use distributed services and resources.</span></span> <span data-ttu-id="94942-197">In manchen Fällen kann es notwendig sein, die Aufgabe eines Vorgangs mithilfe einer kompensierenden Transaktion rückgängig zu machen.</span><span class="sxs-lookup"><span data-stu-id="94942-197">Sometimes, it might be necessary to undo the work performed by an operation by using a compensating transaction.</span></span>

- <span data-ttu-id="94942-198">[Muster „Wiederholung“](./retry.md):</span><span class="sxs-lookup"><span data-stu-id="94942-198">[Retry Pattern](./retry.md).</span></span> <span data-ttu-id="94942-199">Die Ausführung kompensierender Transaktionen kann sich als kostspielig erweisen. Ihre Verwendung kann minimiert werden, indem eine effektive Richtlinie zur Wiederholung fehlerhafter Vorgänge gemäß dem Wiederholungsmuster implementiert wird.</span><span class="sxs-lookup"><span data-stu-id="94942-199">Compensating transactions can be expensive to perform, and it might be possible to minimize their use by implementing an effective policy of retrying failing operations by following the Retry pattern.</span></span>