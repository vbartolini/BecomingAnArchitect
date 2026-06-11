# Lekcja 05 — Infrastructure as Code: Bicep

## Cel lekcji

Po tej lekcji rozumiesz, dlaczego infrastruktura musi być kodem, znasz składnię Bicep w zakresie wystarczającym do opisania realnego systemu (resource, param, module, output), umiesz przejść workflow `what-if` → deploy dla środowisk dev/prod i wiesz, kiedy zamiast Bicep wybrać Terraform. Zakres celowo praktyczny: tyle, ile trzeba do projektu w lekcji 07 — nie pełen kurs IaC.

## Dlaczego to ważne

Dla architekta IaC to nie "narzędzie DevOps", tylko warunek wiarygodności: architektura, której nie da się powtarzalnie odtworzyć, istnieje wyłącznie na slajdzie. W rekrutacjach na role architektoniczne "jak zarządzacie infrastrukturą?" to pytanie-filtr — odpowiedź "klikamy w portalu" kończy temat seniority. A w tym kursie IaC ma rolę bardzo konkretną: w lekcji 07 wdrożysz całą aplikację z modułu 2 jednym poleceniem, i ten fakt ("cała infra w repo, deploy od zera w kilka minut") będzie jednym z najmocniejszych zdań w twoim portfolio.

## Teoria

### Problem: dlaczego klikanie w portalu nie skaluje

Portal Azure jest świetny do nauki i podglądania — i fatalny jako metoda zarządzania infrastrukturą:

1. **Brak powtarzalności.** Środowisko dev wyklikane w 3 godziny — a teraz odtwórz identyczne prod. Z pamięci. Nie pominąwszy żadnego z 40 checkboxów. Każda różnica to przyszły bug klasy "u nas na dev działa".
2. **Drift** — rozjazd między tym, co "powinno być", a tym, co jest. Ktoś hotfixem zmienił ustawienie w portalu o 2 w nocy, nie zapisał nigdzie — od tej chwili nikt nie zna prawdziwego stanu infrastruktury. Bez kodu nie ma nawet definicji "powinno być".
3. **Brak code review i audytu.** Zmiana w aplikacji przechodzi PR, testy, review. Zmiana otwierająca bazę na świat? Jeden klik, zero śladu poza logiem aktywności. IaC zrównuje rygor: infrastruktura przechodzi przez PR jak każdy kod.
4. **Brak dokumentacji.** Szablon IaC **jest** dokumentacją topologii — zawsze aktualną, bo wykonywaną. Diagram może kłamać; szablon nie.

Zasada docelowa: **portal read-only** (podglądanie, debugowanie, nauka), każda zmiana przez kod. W ćwiczeniach lekcji 01–04 klikałeś w portalu — to było celowe (nauka usług); od tej lekcji standard się zmienia.

### ARM → Bicep: co czym jest

Każde wdrożenie w Azure — nawet klik w portalu — przechodzi przez **ARM (Azure Resource Manager)**: warstwę API zarządzającą zasobami. Natywny format ARM to **szablony JSON**: kompletne, ale rozwlekłe i nieprzyjemne w pisaniu (JSON nie ma komentarzy, referencje to stringowe wyrażenia `[resourceId(...)]`).

**Bicep** to język domenowy Microsoftu, który **kompiluje się do ARM JSON** (transpilacja — jak TypeScript do JavaScript). Ta sama moc, ułamek boilerplate'u. Dlaczego Bicep, a nie surowy ARM: zwięzła składnia, komentarze, świetne wsparcie w VS Code (walidacja typów zasobów na żywo), referencje przez symbole zamiast stringów, **day-zero support** — każda nowa funkcja Azure działa w Bicep od premiery, bo Bicep to tylko nakładka na ARM. Istotna cecha modelu: ARM jest **deklaratywny** — opisujesz stan docelowy, a platforma dochodzi do niego (ponowne wdrożenie niezmienionego szablonu nic nie zmienia; to się nazywa idempotencja — termin znasz z messagingu, tu znaczy to samo: wykonanie N razy = wykonanie 1 raz).

### Składnia Bicep na przykładach

**`resource` + `param` + interpolacja** — fundament. Plik `main.bicep`:

```bicep
@description('Środowisko: dev lub prod')
@allowed(['dev', 'prod'])
param environment string

param location string = resourceGroup().location   // wartość domyślna z funkcji

var namespaceName = 'sb-orders-${environment}'      // var = wartość wyliczana

resource serviceBus 'Microsoft.ServiceBus/namespaces@2024-01-01' = {
  name: namespaceName
  location: location
  sku: {
    name: environment == 'prod' ? 'Premium' : 'Standard'
  }
}

// zasób zagnieżdżony: kolejka wewnątrz namespace'u
resource ordersQueue 'Microsoft.ServiceBus/namespaces/queues@2024-01-01' = {
  parent: serviceBus
  name: 'orders'
  properties: {
    maxDeliveryCount: 5          // po 5 nieudanych odbiorach → DLQ (lekcja 02!)
    deadLetteringOnMessageExpiration: true
  }
}
```

Czytaj to tak: `resource <symbol> '<typ>@<wersja-API>'` — typ i wersję podpowiada VS Code (nie ucz się ich na pamięć; wersje API w przykładach tej lekcji zweryfikuj przy pisaniu). Symbol (`serviceBus`) służy do referencji w dalszym kodzie — np. `parent: serviceBus` tworzy zależność, którą ARM sam uwzględni w kolejności wdrażania.

**`module`** — dekompozycja na pliki wielokrotnego użytku. Moduł to po prostu plik `.bicep` z własnymi parametrami:

```bicep
// main.bicep — kompozycja systemu z modułów
module messaging 'modules/servicebus.bicep' = {
  name: 'messaging-deploy'
  params: {
    environment: environment
  }
}

module app 'modules/containerapp.bicep' = {
  name: 'app-deploy'
  params: {
    serviceBusHostName: messaging.outputs.hostName   // output modułu → input innego
  }
}
```

**`output`** — to, co szablon/moduł zwraca na zewnątrz (hostname, ID utworzonej tożsamości). W module `servicebus.bicep`:

```bicep
output hostName string = '${serviceBus.name}.servicebus.windows.net'
```

Outputy spinają moduły między sobą i przekazują wartości do pipeline'u (np. URL aplikacji po wdrożeniu). **Nigdy nie zwracaj outputem sekretów** — outputy są zapisywane jawnie w historii wdrożeń.

I domknięcie lekcji 04 — **RBAC w kodzie**: przypisanie roli to też zasób, więc "consumer ma prawo czytać kolejkę" jest wersjonowane w repo jak wszystko inne:

```bicep
resource sbReceiverRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: ordersQueue                       // least privilege: scope = kolejka
  name: guid(ordersQueue.id, appIdentity.id, 'receiver')
  properties: {
    principalId: appIdentity.properties.principalId
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions',
      '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0')   // Azure Service Bus Data Receiver
    principalType: 'ServicePrincipal'
  }
}
```

### Workflow: az deployment i what-if

Wdrażanie przez Azure CLI (ARM ogarnia kolejność, równoległość i idempotencję):

```bash
az group create --name rg-orders-dev --location westeurope

# 1. Podgląd zmian — co zostanie utworzone/zmienione/usunięte:
az deployment group what-if  --resource-group rg-orders-dev \
  --template-file main.bicep --parameters environment=dev

# 2. Właściwe wdrożenie:
az deployment group create   --resource-group rg-orders-dev \
  --template-file main.bicep --parameters environment=dev
```

**`what-if` to odpowiednik code review wykonany przez maszynę** — diff między stanem faktycznym a szablonem. Nawyk obowiązkowy: nigdy nie wdrażaj na prod bez przeczytania what-if (wyłapuje i twoje pomyłki, i drift po cudzych ręcznych zmianach). W CI/CD what-if wkleja się do PR jako komentarz.

**Środowiska dev/prod:** jeden zestaw szablonów + osobne pliki parametrów (`dev.bicepparam`, `prod.bicepparam`) + osobne resource groups. Różnice między środowiskami (SKU, repliki, retencja) żyją w parametrach i warunkach (`environment == 'prod' ? ... : ...`) — nigdy w skopiowanych szablonach. Kopia szablonu per środowisko to drift w wersji uświęconej.

Praktyczne drobiazgi, które oszczędzą ci godzin: nazwy wielu zasobów (Service Bus namespace, SQL server, Key Vault) muszą być **globalnie unikalne** — standardowy trik to sufiks `uniqueString(resourceGroup().id)`; usunięcie zasobu z szablonu **nie usuwa go z Azure** w domyślnym trybie (incremental) — sprzątaj świadomie; sekretów nie hardkoduj w szablonach — `@secure()` param albo (lepiej) wzorce z lekcji 04.

### Terraform jako alternatywa — kiedy co

**Terraform** (HashiCorp) to najpopularniejsze narzędzie IaC w ogóle: własny język **HCL**, własny **plik stanu** (state file — Terraform pamięta w nim, co utworzył, i diffuje wobec niego; plik trzeba gdzieś bezpiecznie współdzielić, np. w Storage Account) i **providerzy** do setek platform — Azure, AWS, GCP, Kubernetes, GitHub, Cloudflare…

| Kryterium | Bicep | Terraform |
|---|---|---|
| Zasięg | tylko Azure | multi-cloud + SaaS-y |
| Stan | brak pliku stanu (prawdą jest ARM) — mniej ops | state file: potężny (plan, drift detection), ale to dodatkowy artefakt do zarządzania |
| Nowe funkcje Azure | day zero | z opóźnieniem providera (zwykle małym) |
| Rynek pracy | zespoły czysto microsoftowe | szerszy standard branżowy |

**Rekomendacja:** dla tego kursu i stacku w 100% azure'owym — **Bicep** (mniej ruchomych części, zero zarządzania stanem, idealne wsparcie narzędziowe). **Terraform wybierz, gdy:** organizacja jest multi-cloud, IaC ma obejmować zasoby spoza Azure (GitHub, Datadog, Cloudflare), albo zespół platformowy już w nim standaryzuje. Koncepcyjnie to 90% tych samych idei — przejście Bicep→Terraform to dni, nie miesiące; na rozmowie umiejętność nazwania tego trade-offu wystarcza w zupełności.

### Przykład: szkielet systemu z lekcji 07

Struktura, którą rozwiniesz w projekcie — Container App + Service Bus + SQL:

```
infra/
  main.bicep               // kompozycja: resource group scope, moduły, przepływ outputów
  dev.bicepparam
  prod.bicepparam
  modules/
    identity.bicep         // user-assigned managed identity (lekcja 04)
    servicebus.bicep       // namespace + kolejki/topics + role RBAC dla tożsamości
    sql.bicep              // SQL server (Entra-only auth!) + baza serverless
    containerapps.bicep    // środowisko + apki (API, consumer) + KEDA scaling po kolejce
    monitoring.bicep       // Log Analytics + Application Insights (lekcja 07)
```

Zwróć uwagę, jak lekcje się składają: tożsamość z lekcji 04 jest parametrem modułów messaging/sql (RBAC w kodzie), reguła KEDA z lekcji 01 wskazuje kolejkę z lekcji 02, a całość jest jednym `az deployment` z tej lekcji.

## Praktyka

- [ ] Zainstaluj Azure CLI + rozszerzenie Bicep do VS Code. Napisz pierwszy szablon: Storage Account z parametrem `environment`. Przejdź pełny cykl: `what-if` → deploy → zmień coś w szablonie → `what-if` (zobacz diff) → deploy → usuń RG.
- [ ] Zasymuluj drift: wdroż szablon, zmień ręcznie w portalu jakąś właściwość zasobu, uruchom `what-if` — zobacz, jak wykrywa rozjazd. To jest demo, które warto umieć pokazać na rozmowie.
- [ ] Napisz `infra/` dla trójkąta projektu: moduły `servicebus.bicep` (namespace + kolejka z `maxDeliveryCount`), `sql.bicep` (serverless, Entra-only auth), `identity.bicep` (user-assigned MI + role RBAC). Wdróż na dev, sprawdź w portalu, że role assignment istnieje na właściwym scope.
- [ ] Utwórz `dev.bicepparam` i `prod.bicepparam` różniące się SKU Service Busa. Wykonaj `what-if` dla obu — porównaj wyniki. (Prod możesz nie wdrażać — what-if wystarczy i nic nie kosztuje.)
- [ ] Pytanie kontrolne (pisemnie): kiedy NIE warto inwestować w IaC? (Podpowiedź: jednorazowe eksperymenty, nauka usługi w portalu — i co odróżnia te przypadki od "małego projektu, na razie klikamy".)

## Artefakt

Katalog **`infra/`** w repo aplikacji z modułu 2: działające moduły Bicep dla Service Bus + SQL + managed identity z RBAC, parametryzacja dev/prod. To bezpośredni fundament lekcji 07 — tam dojdą Container Apps i monitoring. Commit message w stylu "infra as code: full environment from zero in one command" sam pisze się na post.

## Definition of Done

- [ ] Tłumaczysz bez notatek: czym jest ARM, czym Bicep względem ARM, i 3 powody wyższości IaC nad portalem (drift, powtarzalność, review).
- [ ] Twój szablon przechodzi cykl: czysta resource group → deploy → ponowny deploy bez zmian (nic się nie dzieje — rozumiesz dlaczego) → what-if po ręcznej zmianie w portalu (widzisz drift).
- [ ] `infra/` z modułami, parametrami dev/prod i RBAC w kodzie jest zacommitowane i wdrożone na dev.
- [ ] Umiesz w 2 minuty uzasadnić wybór Bicep vs Terraform dla trzech kontekstów: czysty Azure, multi-cloud, zespół z istniejącym Terraformem.

## Materiały

- [Microsoft Learn — Fundamentals of Bicep](https://learn.microsoft.com/training/paths/fundamentals-bicep/) — oficjalna ścieżka z ćwiczeniami; zrób moduły o szablonach, parametrach i modułach.
- [Bicep documentation — best practices](https://learn.microsoft.com/azure/azure-resource-manager/bicep/best-practices) — krótkie, konkretne; przeczytaj po napisaniu pierwszych szablonów, nie przed.
