# TD : Importer les films populaires depuis TMDB

## Objectifs

À la fin de ce TD, vous saurez :

- déclarer et valider des variables d'environnement dans AdonisJS
- créer une table avec une migration Lucid et en déduire un modèle
- ajouter des propriétés calculées (getters) à un modèle
- consommer une API REST externe depuis un service
- remplir la base de données avec un seeder idempotent

## Point de départ

Récupérer la version : 

```bash
npm install
node ace migration:run
node ace db:seed
npm run dev
```

L'application tourne : on peut s'inscrire et se connecter. Les routes `/movies` et `/movies/:id` existent déjà, mais leurs pages sont vides. Aucun film n'est encore enregistré en base.

## Vue d'ensemble

```
TMDB (API REST)
     │  HTTP + jeton Bearer
     ▼
app/services/tmdb_service.ts     ← client HTTP (ky)
     │
     ▼
database/seeders/movie_seeder.ts ← transforme et enregistre
     │
     ▼
table movies  ⇄  app/models/movie.ts
```

---

## Étape 1 : Obtenir un accès à l'API TMDB

1. Créez un compte sur <https://www.themoviedb.org/signup>.
2. Allez dans **Paramètres → API** et demandez une clé (usage « Developer », projet éducatif).
3. Récupérez deux valeurs :
   - la **clé d'API** (API Key, v3) ;
   - le **jeton d'accès en lecture** (API Read Access Token, v4), une longue chaîne qui commence par `eyJ…`.

> Ces valeurs sont **personnelles et secrètes**. Elles vont dans `.env` (ignoré par Git), jamais dans le code ni dans `.env.example`.

## Étape 2 : Déclarer les variables d'environnement

Une variable d'environnement se déclare à trois endroits :

| Fichier        | Versionné | Rôle                                                  |
| -------------- | --------- | ----------------------------------------------------- |
| `.env`         | non       | valeurs réelles de votre poste, secrets compris       |
| `.env.example` | oui       | modèle pour les autres développeurs, sans les secrets |
| `start/env.ts` | oui       | schéma de validation, vérifié au démarrage            |

AdonisJS valide les variables d'environnement au démarrage. Une variable absente ou mal formée fait échouer le lancement immédiatement, au lieu de provoquer une erreur obscure plus tard.

Nous allons expérimenter deux méthodes : à la main pour les URL de l'API, puis avec la commande `node ace env:add` pour les secrets.

### 2.1 À la main : les URL de l'API

Ajoutez à la fin de `.env.example` :

```dotenv
# TMDB
TMDB_BASE_URL=https://api.themoviedb.org/3
TMDB_IMAGE_BASE_URL=https://image.tmdb.org/t/p

# App name (exposé au client par Vite)
VITE_APP_NAME=Ciné-club
```

Ces valeurs ne sont pas secrètes : on les laisse renseignées dans le modèle. Recopiez les mêmes lignes dans votre `.env`.

Ajoutez ensuite dans l'objet passé à `Env.create`, dans `start/env.ts` :

```ts
  // TMDB
  TMDB_BASE_URL: Env.schema.string({ format: 'url' }),
  TMDB_IMAGE_BASE_URL: Env.schema.string({ format: 'url' }),
```

`{ format: 'url' }` vérifie que la valeur est une URL valide.

### 2.2 Avec `node ace env:add` : les secrets

Lancez la commande sans argument :

```bash
node ace env:add
```

Elle pose trois questions. Répondez pour la clé d'API :

```
Enter the variable name › TMDB_API_KEY
Enter the variable value › <votre clé d'API>
Select the variable type › string
```

Recommencez pour `TMDB_READ_ACCESS_TOKEN` avec votre jeton `eyJ…`.

> Préférez le mode interactif à `node ace env:add TMDB_API_KEY <valeur>` : une valeur passée en argument reste enregistrée dans l'historique du shell.

Ouvrez les trois fichiers et observez ce que la commande a fait pour chaque variable :

- `.env` : ligne ajoutée **avec** la valeur saisie ;
- `.env.example` : ligne ajoutée **sans** valeur (`TMDB_API_KEY=`), le secret ne fuit donc pas dans Git ;
- `start/env.ts` : `TMDB_API_KEY: Env.schema.string()` ajouté à la fin de l'objet.

La commande ne connaît que quatre types (`string`, `number`, `boolean`, `enum`) et ajoute toujours en fin de fichier. Retouchez donc le résultat à la main :

1. dans `.env` et `.env.example`, déplacez les deux lignes sous le commentaire `# TMDB` ;
2. dans `start/env.ts`, déplacez-les sous `// TMDB` et remplacez `Env.schema.string()` par `Env.schema.secret()` :

```ts
  // TMDB
  TMDB_API_KEY: Env.schema.secret(),
  TMDB_READ_ACCESS_TOKEN: Env.schema.secret(),
  TMDB_BASE_URL: Env.schema.string({ format: 'url' }),
  TMDB_IMAGE_BASE_URL: Env.schema.string({ format: 'url' }),
```

`Env.schema.secret()` renvoie un objet `Secret` et non une chaîne. Sa valeur est masquée si on l'affiche dans un `console.log` ou dans un log d'erreur. Pour lire la vraie valeur, il faut appeler `.release()` explicitement.

### 2.3 Bilan des deux méthodes

|                       | À la main                              | `node ace env:add`                       |
| --------------------- | -------------------------------------- | ---------------------------------------- |
| Fichiers modifiés     | un par un, risque d'en oublier un      | les trois en une commande                |
| Valeur dans `.env.example` | au choix                          | toujours vide                            |
| Schéma de validation  | tous (`secret`, `format: 'url'`…)      | `string`, `number`, `boolean`, `enum`    |
| Organisation          | commentaires, regroupement             | ajout en fin de fichier                  |

La commande convient bien aux secrets (valeur vide dans le modèle), on affine ensuite le schéma à la main. Pour une valeur publique ou un format précis, la méthode manuelle reste plus directe.

✅ **Vérification** : retirez temporairement `TMDB_BASE_URL` de `.env` et relancez `npm run dev`. Le serveur refuse de démarrer et indique la variable en cause. Remettez-la.

## Étape 3 : Installer un client HTTP : `ky`

Le projet embarque `axios`, que nous n'utilisons pas. Nous le remplaçons par [`ky`](https://github.com/sindresorhus/ky), plus léger et basé sur l'API `fetch` native de Node.

```bash
npm uninstall axios
npm install ky
```

## Étape 4 : Créer la table `movies`

```bash
node ace make:migration movies
```

Complétez le fichier généré dans `database/migrations/` :

```ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'movies'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id').notNullable()
      table.integer('tmdb_id').notNullable().unique()
      table.string('title').notNullable()
      table.string('original_title').nullable()
      table.text('overview').nullable()
      table.string('poster_path').nullable()
      table.string('backdrop_path').nullable()
      table.date('release_date').nullable()
      table.float('vote_average').notNullable().defaultTo(0)
      table.integer('vote_count').notNullable().defaultTo(0)
      table.float('popularity').notNullable().defaultTo(0)
      table.string('original_language').nullable()

      table.timestamp('created_at').notNullable()
      table.timestamp('updated_at').nullable()
    })
  }

  async down() {
    this.schema.dropTable(this.tableName)
  }
}
```

Points d'attention :

- `id` est **notre** identifiant interne. `tmdb_id` est celui de TMDB. Il est `unique`, ce qui nous permettra de relancer l'import sans créer de doublons (étape 7).
- `overview`, `poster_path`, `release_date`… sont `nullable` : TMDB ne fournit pas toujours ces informations.
- `text` (et non `string`) pour le synopsis, qui peut être long.

Lancez la migration :

```bash
node ace migration:run
```

✅ **Vérification** : ouvrez `database/schema.ts`. Ce fichier est **généré automatiquement** à chaque migration (ne le modifiez jamais à la main). Une classe `MovieSchema` est apparue, avec une propriété typée par colonne, en camelCase (`tmdb_id` → `tmdbId`).

## Étape 5 : Créer le modèle `Movie`

```bash
node ace make:model movie
```

Remplacez le contenu de `app/models/movie.ts` :

```ts
import env from '#start/env'
import { MovieSchema } from '#database/schema'

export default class Movie extends MovieSchema {
  get posterUrl(): string | null {
    return this.posterPath ? `${env.get('TMDB_IMAGE_BASE_URL')}/w500${this.posterPath}` : null
  }

  get backdropUrl(): string | null {
    return this.backdropPath ? `${env.get('TMDB_IMAGE_BASE_URL')}/w1280${this.backdropPath}` : null
  }

  get releaseYear(): number | null {
    return this.releaseDate ? this.releaseDate.year : null
  }
}
```

Le modèle hérite de `MovieSchema` : les colonnes sont déjà déclarées. Le modèle ne contient que la **logique métier**.

TMDB ne renvoie pas d'URL d'image complète, seulement un chemin (`/abc123.jpg`). Pour obtenir l'URL, on ajoute devant la base des images et une taille (`w500` pour une affiche, `w1280` pour une image de fond). Comme la base est stockée dans l'environnement, on peut la changer sans toucher au code.

## Étape 6 : Créer le service TMDB

Un **service** regroupe le code qui parle à un système externe. Les contrôleurs et les seeders l'utilisent sans connaître les détails HTTP.

```bash
node ace make:service tmdb
```

Remplacez le contenu de `app/services/tmdb_service.ts` :

```ts
import env from '#start/env'
import ky, { type KyInstance } from 'ky'

/**
 * https://developer.themoviedb.org/reference/movie-popular-list
 */
export interface TmdbMovie {
  id: number
  title: string
  original_title: string
  overview: string
  poster_path: string | null
  backdrop_path: string | null
  release_date: string
  vote_average: number
  vote_count: number
  popularity: number
  original_language: string
}

export class TmdbService {
  #client: KyInstance

  constructor() {
    this.#client = ky.create({
      prefix: env.get('TMDB_BASE_URL'),
      headers: {
        Authorization: `Bearer ${env.get('TMDB_READ_ACCESS_TOKEN').release()}`,
        Accept: 'application/json',
      },
    })
  }

  /**
   * Récupère une page de films populaires (20 films par page).
   */
  async popular(page: number = 1): Promise<TmdbMovie[]> {
    const response = await this.#client
      .get('movie/popular', { searchParams: { page, language: 'fr-FR' } })
      .json<{ results: TmdbMovie[] }>()

    return response.results
  }
}

export default new TmdbService()
```

À comprendre :

- **`TmdbMovie`** décrit la forme de la réponse JSON de TMDB, qui est en snake_case. TypeScript nous signalera une faute de frappe sur un champ.
- **`ky.create`** crée un client préconfiguré. Toutes les requêtes partagent l'URL de base et l'en-tête `Authorization`.
- **`#client`** est un champ privé natif de JavaScript : inaccessible hors de la classe.
- **`.release()`** extrait la vraie valeur du `Secret` (voir étape 2.2).
- **`export default new TmdbService()`** exporte une instance unique (singleton) prête à l'emploi : `import tmdb from '#services/tmdb_service'`.

✅ **Vérification** dans le REPL :

```bash
node ace repl
```

```js
const { default: tmdb } = await import('#services/tmdb_service')
const movies = await tmdb.popular()
movies.length        // 20
movies[0].title      // un titre en français
```

En cas d'erreur `401 Unauthorized`, revérifiez `TMDB_READ_ACCESS_TOKEN`.

## Étape 7 : Remplir la base avec un seeder

```bash
node ace make:seeder movie
```

Contenu de `database/seeders/movie_seeder.ts` :

```ts
import Movie from '#models/movie'
import tmdb, { type TmdbMovie } from '#services/tmdb_service'
import { BaseSeeder } from '@adonisjs/lucid/seeders'
import { DateTime } from 'luxon'

const PAGES_TO_FETCH = 5

export default class extends BaseSeeder {
  async run() {
    /**
     * Le classement TMDB évolue entre deux appels : un même film peut donc
     * apparaître sur deux pages. On l'indexe par son identifiant TMDB, qui
     * est aussi la clé unique en base.
     */
    const movies = new Map<number, TmdbMovie>()

    for (let page = 1; page <= PAGES_TO_FETCH; page++) {
      for (const movie of await tmdb.popular(page)) {
        movies.set(movie.id, movie)
      }
    }

    const payload = [...movies.values()].map((movie) => ({
      tmdbId: movie.id,
      title: movie.title,
      originalTitle: movie.original_title,
      overview: movie.overview || null,
      posterPath: movie.poster_path,
      backdropPath: movie.backdrop_path,
      releaseDate: movie.release_date ? DateTime.fromISO(movie.release_date) : null,
      voteAverage: movie.vote_average,
      voteCount: movie.vote_count,
      popularity: movie.popularity,
      originalLanguage: movie.original_language,
    }))

    await Movie.updateOrCreateMany('tmdbId', payload)
  }
}
```

Trois idées importantes :

1. **Dédoublonnage avec `Map`** : si un film apparaît deux fois, la seconde écriture remplace la première. On obtient au plus 100 films uniques (5 pages × 20).
2. **Mapping** : on convertit le format de TMDB (snake_case, dates en chaîne, `""` pour un synopsis vide) vers celui de notre modèle (camelCase, `DateTime` Luxon, `null`).
3. **`updateOrCreateMany('tmdbId', …)`** : si un film avec ce `tmdbId` existe, il est **mis à jour**, sinon il est **créé**. Le seeder est donc **idempotent** : on peut le relancer autant de fois qu'on veut pour rafraîchir les notes et la popularité, sans doublons.

Lancez-le :

```bash
node ace db:seed --files database/seeders/movie_seeder.ts
```

✅ **Vérification** :

```bash
node ace repl
```

```js
const { default: Movie } = await import('#models/movie')
await Movie.query().count('* as total')        // ~100
const m = await Movie.query().orderBy('popularity', 'desc').firstOrFail()
m.title, m.posterUrl, m.releaseYear
```

Relancez le seeder puis recomptez : le total ne doit pas avoir changé.

---

## Questions

1. Pourquoi déclarer `TMDB_READ_ACCESS_TOKEN` avec `Env.schema.secret()` plutôt qu'avec `Env.schema.string()` ?
2. Que se passerait-il si on relançait le seeder en remplaçant `updateOrCreateMany` par `createMany` ? Et si la colonne `tmdb_id` n'avait pas de contrainte `unique` ?
3. Pourquoi stocker `poster_path` plutôt que l'URL complète de l'image ?
4. Le seeder appelle l'API page par page, l'une après l'autre. Proposez une version qui lance les 5 requêtes en parallèle. Quel risque cela pose-t-il vis-à-vis d'une API externe ?

## Pour aller plus loin

- Ajoutez au service une méthode `find(tmdbId: number)` qui appelle `GET /movie/{movie_id}` et renvoie le détail d'un film (durée, genres…).
- Ajoutez une colonne `runtime` (durée en minutes) avec une **nouvelle** migration, et complétez-la dans le seeder grâce à `find`.
- Transformez le seeder en commande Ace (`node ace make:command sync_movies`) pour pouvoir rafraîchir le catalogue sans passer par les seeders.
