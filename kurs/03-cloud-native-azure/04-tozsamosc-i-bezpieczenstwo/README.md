# Lekcja 04 — Tożsamość i bezpieczeństwo: Entra ID, managed identities, Key Vault, private endpoints

## Cel lekcji

Po tej lekcji rozumiesz model tożsamości Azure (Entra ID, app registrations, service principals, managed identities), wiesz, co i jak trzymać w Key Vault, znasz podstawy sieci na poziomie potrzebnym architektowi aplikacyjnemu (VNet, private endpoints) — i umiesz zaprojektować system, w którym **w konfiguracji nie ma ani jednego sekretu**.

## Dlaczego to ważne

Wyciek credentiali to wciąż czołowa przyczyna incydentów bezpieczeństwa, a connection string w `appsettings.json` (lub w zmiennej środowiskowej, lub w historii gita…) to jego najpopularniejszy wektor. W on-premise tożsamość "była" — Windows Auth, konta domenowe, integrated security. W chmurze musisz ją **zaprojektować**, i to jest robota architekta, nie "działu security". Na rozmowach pytanie "jak aplikacja łączy się z bazą?" jest testem dojrzałości chmurowej: odpowiedź "connection stringiem z Key Vaulta" jest dobra; odpowiedź "managed identity, bez żadnego sekretu" — bardzo dobra. Ta lekcja prowadzi do tej drugiej.

## Teoria

### Entra ID — katalog tożsamości (od zera)

**Co to jest:** Microsoft Entra ID (do 2023 nazywało się **Azure Active Directory / Azure AD** — w starszych materiałach i SDK wciąż spotkasz starą nazwę) to usługa tożsamości: katalog **principals** (bytów, które mogą się uwierzytelnić) + serwer autoryzacji wydający tokeny OAuth2/OpenID Connect. Każda subskrypcja Azure jest przypięta do jednego **tenanta** Entra ID (tenant = twoja organizacja w katalogu).

Mentalny model dla kogoś z Active Directory: AD on-premise mówiło "kim jesteś w sieci firmowej" (Kerberos, konta domenowe); Entra ID mówi "kim jesteś w świecie HTTP" (tokeny JWT, OAuth2). Analogia jest funkcjonalna, nie techniczna — to różne protokoły i różne modele.

**Rodzaje principals — kluczowe rozróżnienie:**

- **User** — człowiek logujący się hasłem/MFA.
- **Application** — gdy twoja aplikacja sama potrzebuje tożsamości (np. consumer kolejki łączący się z Service Busem), rejestrujesz ją w katalogu. Powstają dwa powiązane obiekty, które wszyscy mylą:
  - **App registration** — *definicja* aplikacji (globalny "szablon": ID, uprawnienia, do czego służy),
  - **Service principal** — *instancja* tej definicji w konkretnym tenancie; to JEMU nadajesz uprawnienia. Analogia: app registration ≈ klasa, service principal ≈ obiekt.

  Klasyczny (i problematyczny) sposób uwierzytelnienia service principala: **client secret** (długi losowy string) albo certyfikat. I tu pies pogrzebany: secret trzeba gdzieś przechowywać, rotować, nie zgubić do gita… Tożsamość mamy, ale problem sekretu wraca tylnymi drzwiami.

### Managed identities — wzorzec kluczowy tej lekcji

**Co to jest:** tożsamość (technicznie: service principal specjalnego typu), którą Azure **sam tworzy i obsługuje dla twojego zasobu compute** — bez żadnego sekretu po twojej stronie. Włączasz managed identity na Container App / Function / App Service, a platforma wystawia temu zasobowi lokalny endpoint, z którego działający kod może pobrać token Entra ID. Nikt — ani ty, ani deweloper, ani pipeline — **nigdy nie widzi credentiala**: nie istnieje hasło, które można wykraść, zgubić, czy zapomnieć zrotować. Rotacja certyfikatów dzieje się w tle, po stronie platformy.

W kodzie .NET to jedna linia, dzięki `DefaultAzureCredential` z pakietu `Azure.Identity`:

```csharp
var client = new ServiceBusClient(
    "mynamespace.servicebus.windows.net",
    new DefaultAzureCredential());
```

`DefaultAzureCredential` próbuje po kolei źródeł tożsamości: na Azure użyje managed identity, a **lokalnie na twojej maszynie — twojego logowania z `az login` lub Visual Studio**. Ten sam kod działa lokalnie i w chmurze, zero konfiguracji warunkowej — to domknięcie wzorca.

**System-assigned vs user-assigned:**

- **System-assigned** — tożsamość przypięta 1:1 do zasobu; żyje i umiera razem z nim (kasujesz Container App → tożsamość znika, a z nią nadane uprawnienia). Prosta, dobra na start i dla pojedynczych zasobów.
- **User-assigned** — tożsamość jako samodzielny zasób, którą podpinasz do wielu zasobów compute. Zalety: jeden zestaw uprawnień dla rodziny usług (np. wszystkie workery systemu), tożsamość przeżywa rebuild zasobu (ważne przy IaC — nie musisz ponownie nadawać ról po każdym `delete + deploy`), można ją utworzyć **przed** zasobem. W projektach z IaC (lekcja 05) user-assigned zwykle wygrywa.

**Ograniczenie:** managed identity działa dla wywołań **usług wspierających Entra ID** (Service Bus, Storage, Key Vault, SQL Database, Cosmos DB…). Dla zewnętrznego API z kluczem licencyjnym sekret nadal istnieje — i właśnie po to jest Key Vault.

### Key Vault — sejf na to, co sekretem być musi

**Co to jest:** zarządzany sejf na **secrets** (stringi: klucze API, hasła do systemów zewnętrznych), **keys** (klucze kryptograficzne do szyfrowania/podpisu) i **certificates**. Dostęp przez API, każdy odczyt audytowany, rozliczanie groszowe per operacja.

**Zmiana myślenia:** w dojrzałym systemie Key Vault to **nie jest** "miejsce na connection stringi do usług Azure" — te eliminujesz managed identities. Key Vault trzyma sekrety, których wyeliminować się nie da: klucze do zewnętrznych API (płatności, SMS-y), credentiale do systemów partnerów, certyfikaty TLS. Heurystyka: *jeśli sekret da się zastąpić managed identity — zastąp; jeśli nie — Key Vault; appsettings/zmienne środowiskowe — nigdy.*

**Jak aplikacja dostaje sekrety — bez kodu po sekret:**
- **Key Vault references** — App Service i Container Apps umieją podstawić wartość z Key Vaulta pod zmienną środowiskową/secret aplikacji (konfiguracja wskazuje URI sekretu, platforma pobiera wartość, używając... managed identity aplikacji — wzorce się składają). Aplikacja czyta zwykłą konfigurację, nieświadoma istnienia sejfu.
- **Provider konfiguracji .NET** (`Azure.Extensions.AspNetCore.Configuration.Secrets`) — sekrety wpadają do `IConfiguration` obok innych źródeł.

A kto pilnuje dostępu do samego sejfu? Entra ID + RBAC — sekretem do sejfu nie jest kolejny sekret, tylko tożsamość. Bez tego mielibyśmy regres w nieskończoność ("gdzie trzymać hasło do miejsca z hasłami?").

### RBAC i least privilege

**RBAC (Role-Based Access Control)** to mechanizm autoryzacji Azure: **przypisanie roli** = kto (principal) + co może (rola = zestaw uprawnień) + gdzie (scope: subskrypcja / resource group / pojedynczy zasób). Przykład docelowy z projektu: *managed identity consumera* + rola *Azure Service Bus Data Receiver* + scope *konkretna kolejka*.

**Least privilege** (zasada najmniejszych uprawnień): nadawaj najwęższą rolę na najwęższym scope, który wystarcza. Antywzorce do wytropienia u siebie: rola **Owner/Contributor** "bo działało" (to role zarządzania zasobami, nie dostępu do danych — consumer kolejki nie potrzebuje umieć skasować namespace'u), scope na całą subskrypcję "bo wygodnie". Zwróć uwagę na podział ról: *control plane* (zarządzanie zasobem: Contributor) vs *data plane* (operacje na danych: `...Data Receiver`, `...Data Sender`, `Key Vault Secrets User`) — aplikacjom prawie zawsze chodzi o te drugie.

### Sieć w pigułce dla architekta nie-sieciowca

Minimum pojęciowe, żeby rozmawiać z zespołem sieciowym i nie projektować systemów otwartych na świat:

- **VNet (Virtual Network)** — twoja prywatna sieć w Azure: przestrzeń adresów IP podzielona na **subnety**. Zasoby w VNet gadają ze sobą prywatnie. **NSG (Network Security Group)** = listy reguł zezwól/odmów na poziomie subnetu — firewall w wersji minimum.
- **Problem:** usługi PaaS (Service Bus, SQL, Key Vault) mają domyślnie **publiczne endpointy** — `twojabaza.database.windows.net` jest osiągalne z internetu (chronione uwierzytelnieniem, ale jednak publiczne). Dla wielu polityk bezpieczeństwa (i audytorów) to za mało.
- **Private endpoint** — rozwiązanie: karta sieciowa w twoim VNet z prywatnym IP, która reprezentuje usługę PaaS. Ruch do bazy płynie prywatnie, a publiczny dostęp do usługi można wyłączyć. Pod spodem działa **Private DNS zone**, dzięki której ta sama nazwa hosta rozwiązuje się do prywatnego IP z wewnątrz VNet — kod aplikacji nie zmienia się ani o znak.
- **Trade-off:** private endpoints kosztują (opłata za endpoint + za przesył) i komplikują development ("dlaczego nie mogę połączyć się z bazą z laptopa?" — bo właśnie o to chodziło; potrzebujesz VPN/Bastion). Dla projektu kursowego wystarczy: publiczne endpointy + Entra-only auth; w architekturze docelowej dla klienta enterprise — private endpoints rysujesz domyślnie. Umiej uzasadnić oba warianty.

### Wzorzec docelowy: zero connection stringów

Składamy klocki w łańcuch, który wdrożysz w lekcji 07:

```
Container App (user-assigned managed identity)
   ── token Entra ID ──► Service Bus   (rola: Data Receiver/Sender, scope: namespace)
   ── token Entra ID ──► Azure SQL     (Entra-only auth; user kontekstowy w bazie utworzony dla tożsamości)
   ── token Entra ID ──► Key Vault     (rola: Secrets User — tylko dla sekretów zewnętrznych)
```

W konfiguracji aplikacji zostają **wyłącznie nazwy hostów** (`mynamespace.servicebus.windows.net`, `myserver.database.windows.net`) — a nazwa hosta nie jest sekretem. Dla SQL connection string przyjmuje postać `Server=...;Authentication=Active Directory Default;` — bez hasła; sterownik pobiera token przez `DefaultAzureCredential`. Audytowalność gratis: każda operacja jest podpisana konkretną tożsamością, a nie anonimowym "kimś, kto znał hasło".

**Kiedy ten wzorzec NIE wystarcza / nie działa:** usługa nie wspiera Entra auth (legacy, zewnętrzni dostawcy) → Key Vault; scenariusze cross-tenant i B2B → wracają service principals z certyfikatami; lokalny development przeciw emulatorom → często jednak connection string (ale tylko lokalnie, w user secrets, nigdy w repo).

## Praktyka

- [ ] Narysuj diagram przepływu tożsamości dla aplikacji z modułu 2: każdy komponent compute, jego tożsamość (system- czy user-assigned? uzasadnij), każda strzałka podpisana rolą RBAC i scope.
- [ ] Ćwiczenie w Azure: utwórz Storage Account + Container App (lub lokalnie z `az login`); nadaj swojej tożsamości rolę *Storage Blob Data Reader* na **kontenerze** (nie subskrypcji!); odczytaj blob z .NET przez `DefaultAzureCredential` — bez żadnego klucza w konfiguracji.
- [ ] Utwórz Key Vault, dodaj sekret (udawany klucz zewnętrznego API), nadaj sobie rolę *Key Vault Secrets User* i odczytaj go z .NET. Potem znajdź wpis o tym odczycie w logach/audycie.
- [ ] Audyt własnej historii: przejrzyj swoje stare projekty (lub repo z modułu 2) pod kątem sekretów w konfiguracji/historii gita. Spisz, co wzorzec z tej lekcji by wyeliminował.
- [ ] Pytanie kontrolne (odpowiedz pisemnie): dlaczego "wszystkie connection stringi trzymamy w Key Vault" to wzorzec przejściowy, a nie docelowy?
- [ ] **Usuń resource group po ćwiczeniach.**

## Artefakt

**Diagram przepływu tożsamości "zero connection stringów"** dla aplikacji z modułu 2 (komponenty, tożsamości, role, scope'y) — wejdzie wprost do projektu w lekcji 07 i do diagramu architektury na Featured. Dobry kandydat na post ("Usunąłem wszystkie connection stringi z konfiguracji — oto jak").

## Definition of Done

- [ ] Tłumaczysz bez notatek różnice: app registration vs service principal vs managed identity; system- vs user-assigned.
- [ ] Wykonałeś z .NET dostęp do co najmniej dwóch usług Azure przez `DefaultAzureCredential`, bez sekretu w konfiguracji.
- [ ] Umiesz powiedzieć, co w twoim systemie trafia do Key Vaulta, a co jest eliminowane przez managed identity — i dlaczego.
- [ ] Wyjaśniasz private endpoint jednym akapitem, łącznie z trade-offem (koszt, dostęp developerski) i tym, kiedy go nie używać.

## Materiały

- [Microsoft Learn — Managed identities for Azure resources](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview) — fundament wzorca tej lekcji.
- [Azure Architecture Center — Security design principles (Well-Architected)](https://learn.microsoft.com/azure/well-architected/security/principles) — szerszy kontekst: identity-first security jako filar Well-Architected Framework.
