# Pinchadiscos

*[English version](README.md)*

*Le dices el ambiente, él te pincha la sesión.*

Aplicación web en Next.js que construye playlists de Spotify a partir de unos cuantos filtros: artistas, géneros, décadas, popularidad y número de canciones. La playlist se puede revisar en el navegador y guardarla después en la cuenta real del usuario.

Proyecto final de la asignatura de Programación Web (UTAD).

## Qué hace

* Login con Spotify mediante OAuth 2.0 (authorization code) con refresco automático del token.
* Widgets de búsqueda para elegir artistas y canciones concretas, con autocompletado sobre la API de Spotify.
* Filtros por género (19 disponibles), por década (de los 70 a los 2020) y por rango de popularidad.
* Generación de la playlist combinando las semillas elegidas, quitando duplicados y aplicando los filtros en local.
* Marcar canciones como favoritas (se guardan en el navegador) para que entren en las siguientes generaciones.
* Añadir más canciones, quitar las que no encajen y guardar el resultado como playlist nueva en Spotify.

## Cómo funciona

Spotify retiró el endpoint de recomendaciones, así que Pinchadiscos no le pide la playlist hecha: lanza varias búsquedas en paralelo a partir de las semillas que eliges, junta todo en un pool y aplica los filtros en local.

```mermaid
flowchart TD
    subgraph seeds["Semillas que elige el usuario"]
        A["Artistas"]
        G["Géneros"]
        C["Canciones sueltas"]
        F["Favoritos<br/>(localStorage)"]
    end

    A --> R1["GET /artists/:id/top-tracks"]
    G --> R2["GET /search?q=genre:..."]
    C --> R3["GET /tracks?ids=..."]
    F --> R3

    R1 --> POOL["Pool de canciones"]
    R2 --> POOL
    R3 --> POOL

    POOL --> DEDUP["Quitar duplicados por id"]
    DEDUP --> P1["Filtro de popularidad<br/>min - max"]
    P1 --> P2["Filtro de década<br/>según release_date"]
    P2 --> CUT["Cortar al límite<br/>30 por defecto"]
    CUT --> UI["Playlist en pantalla<br/>editable"]
    UI --> SAVE["POST /users/:id/playlists<br/>POST /playlists/:id/tracks"]
```

El client secret nunca llega al navegador: el intercambio del código por tokens y el refresco pasan por rutas de servidor.

```mermaid
sequenceDiagram
    participant U as Navegador
    participant S as Servidor Next.js
    participant SP as Spotify

    U->>SP: /authorize (client_id, scopes, state)
    SP-->>U: redirect a /auth/callback?code=...
    U->>S: POST /api/spotify-token (code)
    Note over S: aquí vive el client secret
    S->>SP: POST /api/token (Basic auth)
    SP-->>S: access_token + refresh_token
    S-->>U: solo los tokens
    Note over U: guardados en localStorage

    U->>SP: petición a la API
    SP-->>U: 401, token caducado
    U->>S: POST /api/refresh-token
    S->>SP: grant_type=refresh_token
    SP-->>S: nuevo access_token
    S-->>U: se reintenta la petición original
```

## Stack

Next.js 16 (App Router), React 19, Tailwind CSS 4 y la Spotify Web API. El intercambio de tokens pasa por dos rutas de servidor (`/api/spotify-token` y `/api/refresh-token`) para no exponer el client secret en el navegador.

## Puesta en marcha

Necesitas Node.js 18 o superior y una app creada en el [dashboard de desarrollador de Spotify](https://developer.spotify.com/dashboard).

En la configuración de la app de Spotify, añade esta Redirect URI:

```
http://127.0.0.1:3000/auth/callback
```

Luego, en el proyecto:

```bash
npm install
cp .env.example .env.local
```

Rellena `.env.local` con las credenciales de tu app:

```
SPOTIFY_CLIENT_ID=...
SPOTIFY_CLIENT_SECRET=...
NEXT_PUBLIC_SPOTIFY_CLIENT_ID=...
NEXT_PUBLIC_REDIRECT_URI=http://127.0.0.1:3000/auth/callback
```

Y arranca el servidor:

```bash
npm run dev
```

La app queda en `http://127.0.0.1:3000`. Ten en cuenta que las apps de Spotify en modo desarrollo solo funcionan con las cuentas que añadas manualmente en la sección Users del dashboard.

## Estructura

```
src/
  app/
    page.js               pantalla de login
    dashboard/            pantalla principal con los filtros y la playlist
    auth/callback/        recibe el código de Spotify y lo cambia por el token
    api/                  rutas de servidor para pedir y refrescar tokens
  components/
    widgets/              un componente por filtro
    TrackCard.jsx         tarjeta de canción con favorito, preview y borrado
    PlaylistDisplay.jsx   listado de la playlist generada
  lib/
    auth.js               OAuth, guardado del token y refresco
    spotify.js            llamadas a la API y lógica de generación
    favorites.js          favoritos en localStorage
```

## Limitaciones conocidas

* El filtro de mood (`MoodWidget`) se quedó sin conectar a la generación. Spotify retiró el endpoint de audio features que hacía falta para calcularlo, así que el componente está en el repo pero no se usa.
* La preview de audio depende de `preview_url`, que Spotify ya no devuelve para todas las canciones. Cuando falta, el botón avisa de que no hay preview.
* Los favoritos viven en el `localStorage` del navegador, no en la cuenta de Spotify.

## Licencia

MIT. Ver [LICENSE](LICENSE).
