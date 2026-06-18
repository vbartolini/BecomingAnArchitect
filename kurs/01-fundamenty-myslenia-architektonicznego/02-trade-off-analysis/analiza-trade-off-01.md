**Problem:** Aplikacja do zarządzania czasem, które głównym atutem ma być niezawodność rozumiana poprzez stałą dostępność oraz możliwość szybkiego rozpoczęcia pracy.

**Kryteria (w kolejności priorytetu):** (1) szybka konfiguracja i rozpoczęcie pracy; (2) bez logowania, cookies i zgód (3) możliwość pracy offline (4) koszt utrzymania przez jedną osobę; (5) Odwracalność

| Kryterium | A: Aplikacja webowa z logowaniem w celu przechowywania danych o użytkowniku i jego szablonach dnia/tygodnia pracy | B: Aplikacja webowa zapisująca dane w localstorage, bez logowania | C: Aplikacja desktop |
|---|---|---|---|
| szybka konfiguracja i rozpoczęcie pracy | -- trzeba się zarejestrować, wyrazić zgody, potwierdzić email, prywatność kuleje | ++ uruchamiamy stronę i działamy, na wypadek braku internetu można zainstalować aplikację jako PWA, a dane będzie można wyeksportować do pliku json jako backup | --wymaga ściągnięcia pliku instalacyjnego i wydaje się być to przestarzałym rozwiązaniem. |
| bez logowania, cookies i zgód | -- potrzeba wprowadzenia autoryzacji, zarządzania użytkownikami, powiadomieniami, weryfikacją | ++ zero potrzeb autoryzacji, zarządzania użytkownikami i powiadomieniami | + ma wszystkie zalety opcji B i jedną wadę: w każdym miejscu gdzie korzystamy z aplikacji trzeba ją zainstalować |
| możliwość pracy offline | −− nie da się | ++ PWA rozwiązuje problem | ++ można |
| koszt utrzymania przez jedną osobę | + z pomocą Claude jest to możliwe | + z pomocą Claude jest to możliwe | + z pomocą Claude jest to możliwe |
| Odwracalność | two-way dla danych, ale częściowo one-way dla zobowiązań (RODO, konta, zaufanie) |  two-way (eksport JSON kupuje odwracalność) | two-way technicznie, ale dystrybucja/aktualizacje to osobny koszt zmiany |

**Rekomendacja:** Opcja B. Świadomie płacimy: rezygnujemy z możliwości monitorowania zwyczajów użytkowników w zamian za szybkie ready-to-go użytkownika do pracy. Opcja A odpada przez generowanie problemów z RODO i Privacy oraz zgodami marketingowymi, których potrzeba akceptacji zniechęca do korzystania wielu potencjalnych użytkowników na samym starcie. Opcja C wydaje się być przestarzała, a przede wszystkim uniemożliwia szybkie skorzystanie z narzędzia kiedy nie jest zainstalowane na sprzęcie, z którego korzysta np.: w pracy.
Opcja A byłaby do rozważenia dla użytkowników klasy enterprise, ale obecnie celujemy w osoby, które świadomie chcą zarządzać swoim czasem.