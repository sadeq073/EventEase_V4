# EventEase AI Build Guide (Agents.md 2.0 – volledige herschrijving)

---

## 0. Lees dit eerst

**Publiek:** AI-agents (autonome codegeneratie), menselijke ontwikkelaars, CI-pipelines.

**Gebruik:** Dit document is een *machine-uitvoerbaar runbook*. Alle hoofdsecties bevatten **AI Action Blocks** met:

* **PRE** – vereisten / assumpties die voldaan moeten zijn voordat de taak wordt uitgevoerd.
* **DO** – uit te voeren commando’s / scripts / codegeneratie-instructies.
* **MAKE** – bestanden / klassen / configs die moeten worden aangemaakt of bijgewerkt.
* **CHECK** – tests / validatiestappen / asserts voor idempotentie.
* **ROLLBACK** – minimale herstelactie indien CHECK faalt.

**Taal & Lokalisatie:** Primair Nederlands (`nl`), Engelse fallback (`en`). Commentaar in code mag Engels zijn voor internationale contributies.

**Architectuur-samenvatting:** EventEase is een multitenant event management ERP gebouwd op **Laravel 12.20**, **Laravel Orchid 14.52.3**, **Vue 3 + Inertia.js + Tailwind CSS + DaisyUI**, met **Spatie multitenancy** voor database-isolatie per tenant, **Mollie Cashier** voor abonnements-/betalingsafhandeling, en een modulaire enterprise-architectuur.

**Status (historisch tot 16 juli 2025, Europe/Amsterdam):**

* ✅ Phase 0: Development Foundation.
* ✅ Phase 1: Basisomgeving (frontend stack + multitenancy initial).
* ✅ Phase 2: Orchid integratie & volledige multitenancy patronen.
* ✅ Phase 2.5: Subscription + feature gating (Pennant + plannen).
* ✅ Phase 3: Mollie Cashier Billing geïntegreerd, productie-gereed.
* 🎯 Next Major Milestone: Phase 4 – Uitbreiding Event-domein + API + kalender + rapportages.

---

## 1. Kernprincipes voor AI-bouw

1. **Idempotent bouwen:** Alle scripts moeten veilig meerdere keren kunnen draaien. Controleer of resources bestaan vóór creatie.
2. **Declaratief boven imperatief:** Genereer configuratiebestanden (Composer, NPM, env) als bron van waarheid; voer daarna tooling uit.
3. **Fail fast, log duidelijk:** Artisan-commando’s moeten non-zero exit codes doorgeven; log naar `storage/logs/ai-build.log`.
4. **Geen onnodige interactie:** Gebruik flags (`--force`, `--quiet`) waar veilig, maar nooit om destructieve acties zonder back-up uit te voeren.
5. **Tenant-isolatie absolute eis:** Nooit tenant\_id kolommen in tenant-only tabellen (scheiding gebeurt via DB). Landlord beheert tenants.
6. **Lokale dev ≠ productie:** Documenteer verschillen expliciet; AI mag dev-hulpmiddelen niet naar productie pushen.
7. **Versie-pin cruciale pakketten:** Houd breaking upgrades onder controle; wijzig versies alleen na compatibiliteitscheck.

---

## 2. Versiematrix (minimum getest)

| Component                   | Versie                       | Opmerking                                     |
| --------------------------- | ---------------------------- | --------------------------------------------- |
| PHP                         | 8.4                          | XAMPP/Windows dev; productie Linux aanbevolen |
| Laravel                     | 12.20                        | LTS basis                                     |
| Orchid Platform             | 14.52.3                      | Admin / ERP UI                                |
| spatie/laravel-multitenancy | 4.0.5                        | Database-isolatie                             |
| inertiajs/inertia-laravel   | 2.0.3                        | Backend bridge                                |
| @inertiajs/vue3             | ^2.0.3                       | SPA-ervaring                                  |
| vue                         | ^3.5.17                      | Frontend framework                            |
| tailwindcss                 | ^3.4.0                       | NB: v4 conflicteerde met DaisyUI              |
| daisyui                     | ^5.0.46                      | UI component lib                              |
| vite                        | ^5.4.0                       | Node 18 compat                                |
| laravel/cashier-mollie      | v2.18.0                      | Abonnementen/billing                          |
| laravel/pennant             | latest compat met Laravel 12 | Feature flags                                 |

---

## 3. Projectstructuur (normatief)

```text
q:/XAMPP/htdocs/EventEaseV3/
├── app/
│   ├── Console/Commands/              # Artisan custom (CreateTenantCommand, SetupHostsCommand)
│   ├── Enums/                         # PHP 8.1+ native enums (EventStatus, PlanCodes, ...)
│   ├── Http/
│   │   ├── Controllers/               # DashboardController, BillingController, TenantTestController, ...
│   │   ├── Middleware/                # HandleInertiaRequests, TenantFinderMiddleware, FeatureGateMiddleware
│   │   └── Requests/                  # FormRequests met validatie
│   ├── Models/                        # Tenant, Event, Plan, Subscription, ...
│   ├── Orchid/                        # Platform screens/layouts/filters/presenters
│   │   ├── Screens/
│   │   ├── Layouts/
│   │   ├── Filters/
│   │   └── Presenters/
│   ├── Settings/                      # spatie/laravel-settings classes
│   ├── States/                        # spatie/model-states
│   └── Support/                       # Hulpfuncties, traits (TenantAware)
├── bootstrap/app.php
├── config/
│   ├── multitenancy.php
│   ├── pennant.php
│   ├── cashier_plans.php
│   └── eventease.php                  # (optioneel) centrale projectconfig
├── database/
│   ├── migrations/
│   │   ├── landlord/                  # Landlord-only tabellen (tenants, plannen, billing meta)
│   │   └── *.php                      # Universeel + tenant-conditional
│   ├── seeders/
│   │   ├── LandlordSeeder.php         # plannen, features, demo data landlord
│   │   ├── DemoTenantSeeder.php       # maakt demo tenant + tenant migraties
│   │   └── DatabaseSeeder.php         # orchestrator
├── public/
│   └── index.php                      # Laravel front controller
├── resources/
│   ├── css/app.css
│   ├── js/
│   │   ├── app.js
│   │   ├── Layouts/AppLayout.vue
│   │   └── Pages/Dashboard.vue
│   ├── views/app.blade.php            # Inertia root
│   └── lang/vendor/platform/          # Orchid NL/EN
├── routes/
│   ├── web.php
│   ├── platform.php                   # Orchid admin
│   └── tenant-test.php (DEV ONLY)
├── storage/
├── tailwind.config.js
├── postcss.config.js
├── vite.config.js
├── pint.json
├── phpstan.neon
└── composer.json / package.json
```

---

## 4. Omgevingsmodi

| Mode              | Domein                                      | DB                     | Cache             | Queue     | SSL | Opmerkingen                   |
| ----------------- | ------------------------------------------- | ---------------------- | ----------------- | --------- | --- | ----------------------------- |
| **local (XAMPP)** | `http://localhost` + wildcard `*.localhost` | MySQL via XAMPP        | database          | database  | nee | snelle iteratie               |
| **staging**       | subdomein bv. `staging.eventease.nl`        | MySQL/MariaDB          | redis             | redis     | ja  | demo klanten                  |
| **production**    | klant- of platformdomeinen                  | beheerde MySQL cluster | redis/elasticache | sqs/redis | ja  | schaalbaar, monitoring actief |

---

## 5. Globale Build Volgorde (Top-level Pipeline)

1. **Systeemvereisten controleren** (PHP-extensies, Node, Composer, MySQL bereikbaar).
2. **Repo clonen & branches zetten**.
3. **Composer install** (met versie-pin; optimize-autoload na prod build).
4. **NPM install + build** (dev of prod afhankelijk van mode).
5. **.env genereren** uit sjabloon + lokale overrides.
6. **Applicatiesleutel genereren** (`php artisan key:generate`).
7. **Migrations landlord + universeel** draaien.
8. **Seed landlord data** (plannen, features, admin gebruiker).
9. **DemoTenantSeeder** draaien → maakt tenant + voert tenant migraties.
10. **Orchid admin inrichten** (platform.php routes geverifieerd).
11. **Pennant features laden**; check gating.
12. **Mollie Cashier configureren** (test keys bij local/staging).
13. **Smoke tests (PHPUnit) + larastan + pint**.

Elke stap hieronder heeft een Action Block.

---

## 6. Voorbeeld .env Sjabloon

```properties
APP_NAME=EventEase
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost
APP_LOCALE=nl
APP_FALLBACK_LOCALE=en

# Landlord DB
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=eventeasev3
DB_USERNAME=root
DB_PASSWORD=

# Octane / Performance
OCTANE_SERVER=roadrunner
SESSION_DRIVER=database
CACHE_STORE=database
QUEUE_CONNECTION=database

# Mollie (test)
MOLLIE_KEY=test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Sanctum / SPA
SANCTUM_STATEFUL_DOMAINS=localhost,127.0.0.1
SESSION_DOMAIN=localhost

# Multitenancy
TENANT_DB_PREFIX=eventease_tenant_
```

---

## 7. Composer Kernpackages (composer.json fragment)

```json
{
  "require": {
    "php": "^8.4",
    "laravel/framework": "^12.20",
    "orchid/platform": "^14.52.3",
    "spatie/laravel-multitenancy": "^4.0.5",
    "inertiajs/inertia-laravel": "^2.0.3",
    "laravel/sanctum": "^5.0",
    "spatie/laravel-data": "^4.0",
    "spatie/laravel-model-states": "^3.0",
    "spatie/laravel-tags": "^5.0",
    "spatie/laravel-settings": "^3.0",
    "spatie/laravel-activitylog": "^5.0",
    "spatie/laravel-backup": "^8.0",
    "spatie/laravel-webhook-client": "^4.0",
    "laravel/pennant": "^1.0",
    "mollie/laravel-cashier-mollie": "^2.18",
    "maatwebsite/excel": "^4.0",
    "spatie/browsershot": "^3.0"
  },
  "require-dev": {
    "barryvdh/laravel-ide-helper": "^3.0",
    "larastan/larastan": "^3.0",
    "laravel/pint": "^1.0",
    "phpunit/phpunit": "^11.0"
  }
}
```

---

## 8. NPM Stack (package.json fragment)

```json
{
  "private": true,
  "scripts": {
    "dev": "vite --host",
    "build": "vite build",
    "lint": "eslint . --ext .js,.vue"
  },
  "devDependencies": {
    "vite": "^5.4.0",
    "@vitejs/plugin-vue": "^4.6.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "latest",
    "postcss": "latest"
  },
  "dependencies": {
    "@inertiajs/vue3": "^2.0.3",
    "vue": "^3.5.17",
    "daisyui": "^5.0.46"
  }
}
```

---

## 9. Tailwind + DaisyUI Config (EventEase thema)

```js
// tailwind.config.js
module.exports = {
  content: [
    "./vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php",
    "./vendor/orchid/platform/resources/views/**/*.blade.php",
    "./storage/framework/views/*.php",
    "./resources/views/**/*.blade.php",
    "./resources/js/**/*.vue",
  ],
  plugins: [require('daisyui')],
  daisyui: {
    themes: [
      "light", "dark", "corporate", "business",
      {
        "eventease": {
          "primary": "#1d4ed8",
          "secondary": "#f000b8",
          "accent": "#37cdbe",
          "neutral": "#3d4451",
          "base-100": "#ffffff",
        }
      }
    ]
  }
}
```

---

## 10. Vite Config

```js
// vite.config.js
import { defineConfig } from 'vite'
import laravel from 'laravel-vite-plugin'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [
    laravel({
      input: ['resources/css/app.css', 'resources/js/app.js'],
      refresh: true,
    }),
    vue({
      template: { transformAssetUrls: { base: null, includeAbsolute: false } },
    }),
  ],
  resolve: { alias: { vue: 'vue/dist/vue.esm-bundler.js' } },
})
```

---

## 11. Multitenancy – Conceptueel

**Landlord DB** beheert tenants, plannen, facturatie. **Tenant DB** bevat domeinspecifieke data (events, bezoekers, registraties). Scheiding op database-niveau; geen `tenant_id` kolommen in tenant-only tabellen. Context-activatie: `Tenant::makeCurrent()` (Spatie).

**Belangrijke patronen:**

* Landlord-only migraties in `database/migrations/landlord/`.
* Universele migraties draaien overal (users, roles indien gedeeld schema gewenst; in strikte scheiding kunnen users tenant-only zijn – kies strategie per projectfase).
* Tenant-only migraties conditioneel binnen migratiebestand: `if (app()->bound('currentTenant')) { ... }`.
* Tenant migraties uitvoeren via Spatie `MigrateTenantAction` of `tenants:artisan` wrapper.

---

## 12. Tenant Model (EventEase variant)

```php
<?php
namespace App\Models;

use Spatie\Multitenancy\Models\Tenant as BaseTenant;
use Laravel\Cashier\Billable; // Mollie Cashier

class Tenant extends BaseTenant
{
    use Billable; // voor abonnementen

    protected $fillable = ['name', 'domain', 'database', 'email'];

    public function getDatabaseName(): string
    {
        // Prefix uit .env instelbaar; fallback op kolom
        return config('multitenancy.tenant_database_prefix', 'eventease_tenant_') . $this->id;
    }

    public function isActive(): bool
    {
        // Uit te breiden met abonnementstatus / soft delete / blokkering
        return true;
    }

    public function getUrl(): string
    {
        return 'https://' . $this->domain;
    }

    /* ===== Billing helpers ===== */
    public function mollieCustomerFields(): array
    {
        return [ 'email' => $this->email, 'name' => $this->name ];
    }

    public function getInvoiceInformation(): array
    {
        return [$this->name, $this->email, $this->domain];
    }

    public function getCurrentSubscription() { return $this->subscriptions()->where('status','active')->first(); }
    public function getCurrentPlan() { $s=$this->getCurrentSubscription(); return $s?Plan::where('code',$s->plan_code)->first():null; }
    public function hasActiveAccess(): bool { return (bool) $this->getCurrentSubscription(); }
}
```

---

## 13. TenantFinderMiddleware (vereenvoudigd, DEV wildcard)

> **LET OP:** In productie GEEN `*.localhost` wildcard.

```php
public function handle($request, Closure $next)
{
    $host = $request->getHost();

    // Ontwikkeling: map *.localhost naar tenant domeinrecord met zelfde subdomein
    if (str_ends_with($host, '.localhost')) {
        $sub = substr($host, 0, -strlen('.localhost'));
        $tenant = Tenant::where('domain', $sub . '.localhost')->orWhere('domain', $sub . '.eventease.test')->first();
    } else {
        $tenant = Tenant::where('domain', $host)->first();
    }

    if ($tenant) {
        $tenant->makeCurrent();
    }

    return $next($request);
}
```

---

## 14. DemoTenantSeeder

```php
<?php
class DemoTenantSeeder extends Seeder
{
    public function run(): void
    {
        $demoTenant = Tenant::firstOrCreate(
            ['domain' => 'demo.eventease.test'],
            ['name' => 'Demo EventEase', 'database' => 'eventease_demo', 'email' => 'demo@example.test']
        );

        $demoTenant->makeCurrent();
        $this->seedTenantData();
    }

    private function seedTenantData(): void
    {
        $tenant = app('currentTenant');
        if ($tenant) {
            app(\Spatie\Multitenancy\Actions\MigrateTenantAction::class)
                ->execute($tenant);
        }
        // Tenant seed logica hier
        \Log::info('DemoTenantSeeder: tenant data seeded.');
    }
}
```

---

## 15. Voorbeeld Tenant-only Migratie (Events)

```php
return new class extends Migration {
    public function up(): void
    {
        if (app()->bound('currentTenant') && !Schema::hasTable('events')) {
            Schema::create('events', function (Blueprint $table) {
                $table->id();
                $table->string('name');
                $table->text('description')->nullable();
                $table->dateTime('start_date');
                $table->dateTime('end_date');
                $table->string('location')->nullable();
                $table->integer('max_attendees')->nullable();
                $table->decimal('price', 8, 2)->default(0);
                $table->json('settings')->nullable();
                $table->boolean('is_active')->default(true);
                $table->timestamps();
                $table->index(['is_active','start_date']);
            });
        }
    }
    public function down(): void
    {
        if (app()->bound('currentTenant')) {
            Schema::dropIfExists('events');
        }
    }
};
```

---

## 16. Landlord-only Migratie (Tenants)

```php
Schema::create('tenants', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('domain')->unique();
    $table->string('database');
    $table->string('email')->nullable();
    $table->timestamps();
});
```

---

## 17. Universele Migratie (Users)

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
```

---

## 18. Frontend – Basiscomponenten

### AppLayout.vue

```vue
<template>
  <div class="min-h-screen bg-base-200">
    <div class="navbar bg-base-100 shadow-lg">
      <Link :href="'/admin'" class="btn btn-ghost text-xl">
        <span class="font-bold text-primary">Event</span><span class="font-light">Ease</span>
      </Link>
    </div>
    <main class="container mx-auto p-4">
      <slot />
    </main>
    <footer class="footer footer-center p-10 bg-base-100">© {{ new Date().getFullYear() }} EventEase</footer>
  </div>
</template>
<script setup>
import { Link } from '@inertiajs/vue3'
</script>
```

### Dashboard.vue (voorbeeld)

```vue
<template>
  <AppLayout title="Dashboard">
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      <div class="card bg-base-100 shadow-xl">
        <div class="card-body">
          <h2 class="card-title text-primary">{{ stats.totalEvents }} Events</h2>
        </div>
      </div>
    </div>
  </AppLayout>
</template>
<script setup>
import AppLayout from '@/Layouts/AppLayout.vue'
const props = defineProps({ stats: Object, recentActivity: Array })
</script>
```

---

## 19. Inertia Controllerpatroon

```php
class DashboardController extends Controller
{
    public function index()
    {
        return Inertia::render('Dashboard', [
            'stats' => [
                'totalEvents' => Event::count(),
                'activeTenants' => Tenant::count(),
                'tenantGrowth' => 25,
                'newRegistrations' => 12,
            ],
            'recentActivity' => Activity::latest()->take(5)->get(),
        ]);
    }
}
```

---

## 20. Inertia Middleware (gedeelde data)

```php
public function share(Request $request): array
{
    return [
        ...parent::share($request),
        'auth' => $request->user() ? [
            'id' => $request->user()->id,
            'name' => $request->user()->name,
            'email' => $request->user()->email,
        ] : null,
        'flash' => [
            'message' => fn () => $request->session()->get('message'),
            'error' => fn () => $request->session()->get('error'),
        ],
        'app' => [
            'name' => config('app.name'),
            'version' => '3.0.0',
            'environment' => config('app.env'),
        ],
    ];
}
```

---

## 21. Abonnementen & Feature Gating

### Plans (config/cashier\_plans.php)

```php
return [
  'plans' => [
    'basic' => [ 'amount' => ['value'=>'0.00','currency'=>'EUR'], 'interval'=>'1 month', 'description'=>'Basic Plan - Perfect for getting started'],
    'pro' => [ 'amount' => ['value'=>'49.00','currency'=>'EUR'], 'interval'=>'1 month', 'description'=>'Pro Plan - Advanced features for professionals'],
    'premium' => [ 'amount' => ['value'=>'99.00','currency'=>'EUR'], 'interval'=>'1 month', 'description'=>'Premium Plan - Enterprise-grade features'],
  ],
];
```

### Pennant Featuredefinities (TenantFeatures)

```php
class TenantFeatures
{
    const EVENTS = 'events';
    const REPORTING = 'reporting';
    const API = 'api';
    const BILLING = 'billing';
    // ... overige
}
```

Koppel features aan plannen via LandlordSeeder + pivot-tabellen (`plans`, `features`, `plan_feature`).

### FeatureGateMiddleware

* Controleer currentTenant → currentPlan → features.
* Indien geen toegang: redirect + flash met upgrade-aanbod.

---

## 22. Mollie Cashier Integratie (samenvatting)

**Belangrijk:** Gebruik *Mollie test key* in local/staging. Productie vereist HTTPS en geconfigureerde webhook.

### Config stappen

1. `composer require mollie/laravel-cashier-mollie:^2.18`.
2. Publiceer config & migraties: `php artisan vendor:publish --tag="cashier-mollie-config"` etc.
3. Voeg Billable trait toe aan Tenant (zie §12).
4. Configureer plannen in `config/cashier_plans.php`.
5. Milieuvariabele: `MOLLIE_KEY`.
6. Webhook route beveiligen (`Route::post('/mollie/webhook', ...)`).

### Facturatieflow

* Bij plan-keuze → createOrGetMollieCustomer → nieuwe subscription → Mollie checkout.
* Bij succesvolle betaling → webhook → status update subscription → features vrijgeven.

---

## 23. Orchid Platform Integratie

**Waarom:** Snelle enterprise admin, ACL, scherm-gebaseerde UI.

### Minimale eerste custom screen

```php
class EventListScreen extends Screen
{
    public $name = 'Events';
    public $description = 'Beheer events';

    public function query(): iterable
    {
        return [ 'events' => Event::filters()->defaultSort('id')->paginate() ];
    }

    public function layout(): iterable
    {
        return [ EventFiltersLayout::class, EventListLayout::class ];
    }
}
```

### Branding

* Voeg EventEase thema CSS / logo toe via Orchid publish assets of custom layout.
* Lokalisatiebestanden NL/EN in `resources/lang/vendor/platform/`.

---

## 24. Teststrategie

### Testcommando’s

```bash
php artisan test                 # alle tests
php artisan test --coverage      # coverage rapport
php artisan test tests/Feature/EventManagementTest.php
vendor/bin/phpstan analyse       # statische analyse
vendor/bin/pint                  # code style
```

### Voorbeeld Feature Test

```php
class EventManagementTest extends TestCase
{
    use RefreshDatabase;

    public function test_admin_can_create_event(): void
    {
        $admin = User::factory()->create();
        $this->actingAs($admin);

        $response = $this->post('/admin/events', [
            'title' => 'Test Event',
            'description' => 'Test Description',
            'start_date' => now()->addDays(7),
            'end_date' => now()->addDays(8),
        ]);

        $response->assertRedirect();
        $this->assertDatabaseHas('events', ['title' => 'Test Event']);
    }
}
```

---

## 25. Security Checklist

* ✅ Alle admin routes achter Orchid auth.
* ✅ Sanctum tokens voor SPA/API.
* ✅ Rol-/rechtenmodel via Orchid ACL.
* ✅ Vraagvalidatie (Request classes) met duidelijke regels.
* ✅ CSRF actief; check multi-domein SANCTUM\_STATEFUL\_DOMAINS.
* ✅ Encrypt gevoelige billing-data.

**Validatievoorbeeld:**

```php
$request->validate([
  'title' => 'required|string|max:255',
  'description' => 'nullable|string|max:5000',
  'start_date' => 'required|date|after:now',
  'end_date' => 'required|date|after:start_date',
]);
```

---

## 26. Performance & Schaling

* Eager loading relaties.
* Indexeer veelgebruikte kolommen (`is_active`, `start_date`).
* Cache zware queries (per tenant cache prefix!).
* Redis aanbevolen voor cache/queue productie.
* Octane (RoadRunner op Windows, Swoole/rr in Linux prod) – optioneel.

---

## 27. Backup & Monitoring

* spatie/laravel-backup configureren → DB + bestanden + cloud opslag.
* Laravel Pulse voor metrics.
* Log rotation + externe shipping (bijv. Logtail / ELK).
* Payment events loggen (Mollie webhooks).

---

## 28. Apache/XAMPP Dev Config (NIET PROD)

```apache
<VirtualHost *:80>
    DocumentRoot "Q:/XAMPP/htdocs/EventEaseV3/public"
    ServerName localhost
    ServerAlias *.localhost
    <Directory "Q:/XAMPP/htdocs/EventEaseV3/public">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

**Hosts file (Windows):**

```
127.0.0.1 demo.localhost
127.0.0.1 demo.eventease.test
```

---

## 29. Development Hulpmiddelen

### Artisan Helpers

| Commando                                           | Doel                              |
| -------------------------------------------------- | --------------------------------- |
| `php artisan tenant:create "Naam" "sub.localhost"` | Maak tenant + DB                  |
| `php artisan tenants:artisan "migrate"`            | Migraties op alle tenants         |
| `php artisan setup:hosts`                          | Toon vereiste hosts entries (DEV) |
| `php artisan test:tenant`                          | Controleer tenant-switch          |

### NPM Build

| Commando        | Resultaat                                       |
| --------------- | ----------------------------------------------- |
| `npm run dev`   | Vite HMR op `http://localhost:5173`             |
| `npm run build` | Productiebundel (JS \~300KB gz, CSS \~100KB gz) |

---

## 30. Smoke Test Scenario’s

**ST01 – Dashboard Render:**

* URL: `/test-dashboard` (landlord context).
* Verwacht: Vue component laadt, DaisyUI thema actief.

**ST02 – Tenant Dashboard:**

* URL: `http://demo.localhost/test-dashboard`.
* Verwacht: `currentTenant` geactiveerd, events tellen uit tenant DB.

**ST03 – Billing Screen:**

* URL: `/admin/billing`.
* Verwacht: huidig plan zichtbaar; upgrade knoppen renderen.

**ST04 – Feature Gate:**

* Forceer tenant op Basic → bezoek feature-route Pro → redirect + upgrade melding.

---

## 31. Productie Deploy Checklist

* [ ] Productie-domeinen en SSL-certificaten actief.
* [ ] `.env` → `APP_ENV=production`, `APP_DEBUG=false`.
* [ ] Redis geconfigureerd: `CACHE_STORE=redis`, `QUEUE_CONNECTION=redis`, `SESSION_DRIVER=redis`.
* [ ] MySQL prod credentials ingesteld.
* [ ] Mollie live sleutel (`MOLLIE_KEY=live_xxx`).
* [ ] Webhook URL geregistreerd in Mollie dashboard.
* [ ] Backups geconfigureerd + test restore.
* [ ] Development-only routes en wildcard \*.localhost verwijderd.
* [ ] Config cache + route cache + event cache draaien.
* [ ] Supervisor/queue workers opzetten.

---

## 32. AI Action Blocks

Onderstaande blokken kun je als AI letterlijk uitvoeren in volgorde. Vervang variabelen tussen `{{ ... }}`.

### AIB-01: Controleer systeemvereisten

**PRE:** Shell-toegang; PHP/Composer/Node moeten geïnstalleerd zijn.
**DO:**

```bash
php -v
composer -V
node -v
npm -v
mysql --version || mariadb --version
```

**CHECK:** PHP ≥8.4; Node ≥18; MySQL bereikbaar.
**ROLLBACK:** Installeer ontbrekende afhankelijkheden; stop.

---

### AIB-02: Repo clonen

**DO:**

```bash
git clone https://github.com/sadeq073/EventEase_V3.git eventease
cd eventease
git checkout dev || git checkout -b dev origin/dev || git checkout main
```

**CHECK:** map aanwezig; `composer.json` gevonden.

---

### AIB-03: Composer install

```bash
composer install --no-interaction --prefer-dist
```

**CHECK:** `vendor/autoload.php` bestaat.

---

### AIB-04: NPM install & build

```bash
npm ci || npm install
npm run build || npm run dev
```

**CHECK:** `public/build/manifest.json` of Vite dev running.

---

### AIB-05: .env genereren

```bash
cp .env.example .env || printf "APP_KEY=\n" > .env
php artisan key:generate --force
```

**CHECK:** `APP_KEY` gevuld.

---

### AIB-06: Migrate landlord + universeel

```bash
php artisan migrate:fresh --force
php artisan migrate --path=database/migrations/landlord --force
```

**CHECK:** landlord tabellen (`tenants`, plannen) bestaan.

---

### AIB-07: Seed landlord

```bash
php artisan db:seed --class=LandlordSeeder --force
```

**CHECK:** plannen & features tabellen gevuld; admin user bestaat.

---

### AIB-08: Demo tenant aanmaken

```bash
php artisan db:seed --class=DemoTenantSeeder --force
```

**CHECK:** demo tenant entry; tenant DB gecreëerd; tenant migraties uitgevoerd (events tabel).

---

### AIB-09: Tenant handmatig

```bash
php artisan tenant:create "{{Company Name}}" "{{sub.domain}}"
```

**CHECK:** DB aangemaakt; tenant migraties success.

---

### AIB-10: Test multitenancy

```bash
php artisan tenants:artisan "migrate:status"
php artisan tenants:artisan "tinker --execute='echo DB::connection()->getDatabaseName();'"
```

---

### AIB-11: Billing test (Mollie sandbox)

**PRE:** `MOLLIE_KEY` test.
**DO:** via web UI upgrade tenant naar Pro; volg redirect naar Mollie test checkout.
**CHECK:** webhook verwerkt → subscription status active.

---

## 33. Troubleshooting Matrix

| Symptoom                                   | Mogelijke oorzaak              | Diagnose                                           | Oplossing                                                   |
| ------------------------------------------ | ------------------------------ | -------------------------------------------------- | ----------------------------------------------------------- |
| "Base table or view not found: events"     | Tenant context niet actief     | dump `app('currentTenant')`                        | Activeer TenantFinderMiddleware / gebruik `->makeCurrent()` |
| Tailwind build error `./base not exported` | Tailwind v4 + DaisyUI          | `npm list tailwindcss`                             | Downgrade naar ^3.4.0                                       |
| Vite crash `crypto.hash is not a function` | @vitejs/plugin-vue on Node 18  | `npm ls @vitejs/plugin-vue`                        | Gebruik ^4.6.0                                              |
| Mollie webhooks niet verwerkt              | Geen publieke URL              | Check logs                                         | Gebruik ngrok of staging domein                             |
| 500 op admin                               | Orchid assets niet gepublished | `php artisan vendor:publish --tag=platform-config` | Publish opnieuw                                             |

---

## 34. Lokalisatie

* Alle user-facing strings via `__('...')`.
* NL primair; EN fallback.
* Orchid vendor publicatie → overschrijf NL vertalingen.
* Controle: `app()->getLocale()`.

---

## 35. Codekwaliteit & CI-hooks

### pre-commit (suggestie)

```bash
#!/usr/bin/env bash
vendor/bin/pint --dirty || exit 1
vendor/bin/phpstan analyse || exit 1
php artisan test --parallel --retries=1 || exit 1
```

---

## 36. Release Workflow (Samenvatting)

1. Merge feature branch → `dev`.
2. CI: lint/test/analyse/build → artefact.
3. Handmatige QA in staging.
4. Tag release `vX.Y.Z` → merge naar `main`.
5. Productiedeploy via pipeline (env secrets, migraties, cache).

---

## 37. Roadmap Phase 4 (aanbevolen volgende stappen)

1. **Event CRUD voltooien** (registratie, tickets, statussen, workflow-states).
2. **Kalenderweergave** (FullCalendar of custom; ICS export).
3. **REST/GraphQL API** (tenant-auth via Sanctum tokens).
4. **Rapportages & exports** (maatwebsite/excel, Browsershot PDF).
5. **Activiteitenlog UI** (spatie/activitylog).
6. **Notificaties** (mail, web, webhook-client integraties).

---

## 38. Referentie URLs (DEV)

* Hoofd: `http://localhost/` (redirect → /admin)
* Admin (Orchid): `http://localhost/admin`
* Test dashboard: `http://localhost/test-dashboard`
* Demo tenant dashboard: `http://demo.localhost/test-dashboard`
* Vite HMR: `http://localhost:5173`

---

## 39. Bewaarpunten & Back-up Aanpak

* Dagelijkse landlord DB dump.
* Nachtelijke loop over tenants → dump per DB.
* Bestandsback-up: `storage/app`, `public/uploads`.
* Versleutel off-site opslag.

---

## 40. Minimale Data Flow (tekstueel sequence diagram)

```
Browser → Apache → Laravel (landlord) → TenantFinder? ja → switch DB → Controller → Model (tenant DB) → Inertia → Vue component → UI
Billing upgrade → Tenant → Mollie API → Webhook → Laravel callback → update subscription → features vrijgeven
```

---

## 41. Snelle Evaluatie voor Nieuwe Contributies (AI Checklist)

* [ ] Past code in gedefinieerde projectstructuur?
* [ ] Houdt rekening met tenant context? (geen cross-DB queries!)
* [ ] Bevat NL/EN vertaalbare tekst?
* [ ] Heeft tests?
* [ ] Geformatteerd door Pint?
* [ ] Statische analyse groen?
* [ ] Performance impact beoordeeld?

---

## 42. Licentie & Credits

Projectbron: [https://github.com/sadeq073/EventEase\_V3.git](https://github.com/sadeq073/EventEase_V3.git) (controleer licentie in repo).

---

## 43. Appendix – Volledige Installatie Script (Linux bash, voorbeeld)

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_DIR=${1:-/var/www/eventease}

if [ ! -d "$PROJECT_DIR" ]; then
  git clone https://github.com/sadeq073/EventEase_V3.git "$PROJECT_DIR"
fi
cd "$PROJECT_DIR"

git fetch --all
git checkout dev || git checkout main

composer install --no-interaction --prefer-dist --optimize-autoloader

cp -n .env.example .env || true
php artisan key:generate --force

npm ci || npm install
npm run build

php artisan migrate:fresh --force
php artisan migrate --path=database/migrations/landlord --force
php artisan db:seed --class=LandlordSeeder --force
php artisan db:seed --class=DemoTenantSeeder --force

php artisan optimize
```

---

## 44. Appendix – Windows PowerShell Quickstart
```powershell
$proj = "Q:\XAMPP\htdocs\EventEaseV3"
if (!(Test-Path $proj)) { git clone https://github.com/sadeq073/EventEase_V3.git $proj }
Set-Location $proj

git checkout dev
composer install --no-interaction
Copy-Item .env.example .env -ErrorAction SilentlyContinue
php artisan key:generate --force

npm install
npm run build

php artisan migrate:fresh --force
php artisan migrate --path=database/migrations/landlord --force
php artisan db:seed --class=LandlordSeeder --force
php artisan db:seed --class=DemoTenantSeeder --force
```

---

## 45. Slot
> **Contributieregel:** Update eerst deze gids → voer build scripts uit → commit code + gidswijziging samen.
**Laatste herziening:** 16 juli 2025 (Europe/Amsterdam).
