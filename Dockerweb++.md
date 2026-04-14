```bash
meow@debserv:~/projet_cyber$ mkdir projet-web-stats && cd projet-web-stats

meow@debserv:~/projet_cyber/projet-web-stats$ mkdir src && cd src
```

```bash
nano Dockerfile
```
```bash
FROM php:8.2-apache
RUN docker-php-ext-install pdo pdo_mysql
COPY . /var/www/html/
RUN chown -R www-data:www-data /var/www/html
```
```bash
cd ..
nano docker-compose.yml
```
```bash

version: '3.8'

services:
  web-interface:
    build: ./src
    container_name: web-dmz
    ports:
      - "8080:80"
    depends_on:
      - bdd-service

  bdd-service:
    image: mariadb:latest
    container_name: mysql-lan
    environment:
      MYSQL_ROOT_PASSWORD: root_password_fort
      MYSQL_DATABASE: backup_infra_db
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
```
```bash
cd src
nano config.php

```

```bash 
<?php
define('DB_HOST', 'bdd-service');
define('DB_PORT', '3306');
define('DB_NAME', 'backup_infra_db');
define('DB_USER', 'user_web');
define('DB_PASS', 'PassWeb123!');
define('AFFICHER_ERREURS_DB', true);
define('LIMITE_RESULTATS', 10);
define('DB_DSN', 'mysql:host=' . DB_HOST . ';port=' . DB_PORT . ';dbname=' . DB_NAME . ';charset=utf8mb4');


```
```bash
nano index.php
```
```bash

<?php
/**
 * =====================================================
 *  SCRIPT PRINCIPAL - Affichage des données clients
 *  Serveur Web (DMZ) → Base de données (LAN)
 *  Accessible depuis les clients WAN
 * =====================================================
 */

declare(strict_types=1);

// ─── SÉCURITÉ HTTP ────────────────────────────────────────────────────────────
header('X-Frame-Options: DENY');
header('X-Content-Type-Options: nosniff');
header('X-XSS-Protection: 1; mode=block');
header('Content-Type: text/html; charset=UTF-8');

// ─── CHARGEMENT DE LA CONFIGURATION ─────────────────────────────────────────
require_once __DIR__ . '/config.php';

// ─── GESTION DES ERREURS ─────────────────────────────────────────────────────
if (AFFICHER_ERREURS_DB) {
    ini_set('display_errors', '1');
    error_reporting(E_ALL);
} else {
    ini_set('display_errors', '0');
    error_reporting(0);
}

// ─── FONCTIONS UTILITAIRES ───────────────────────────────────────────────────

/**
 * Connexion PDO sécurisée au serveur de base de données (LAN)
 */
function connecterDB(): PDO
{
    $options = [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        PDO::ATTR_EMULATE_PREPARES   => false,
        PDO::ATTR_TIMEOUT            => 5, // timeout connexion (secondes)
    ];

    return new PDO(DB_DSN, DB_USER, DB_PASS, $options);
}

/**
 * Échappe proprement une valeur pour l'affichage HTML
 */
function h(mixed $val): string
{
    return htmlspecialchars((string)($val ?? ''), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

/**
 * Récupère les données d'un client depuis la DB
 * (utilisation de requêtes préparées pour éviter les injections SQL)
 *
 * @param PDO    $pdo        Instance de connexion
 * @param string $recherche  Terme de recherche (nom ou email)
 * @return array             Tableau de résultats
 */
function rechercherClients(PDO $pdo, string $recherche = ''): array
{
    // On adapte aux colonnes réelles : id_client, nom_client, dossier_racine
    $sql = "SELECT
                id_client as id,
                nom_client as nom,
                dossier_racine as dossier,
                duree_conservation as conservation
            FROM clients";

    if ($recherche !== '') {
        $sql .= " WHERE nom_client LIKE :recherche";
        $sql .= " ORDER BY nom_client ASC LIMIT 10" . LIMITE_RESULTATS;
        $stmt = $pdo->prepare($sql);
        $stmt->execute([':recherche' => '%' . $recherche . '%']);
    } else {
        $sql .= " ORDER BY nom_client ASC LIMIT " . LIMITE_RESULTATS;
        $stmt = $pdo->query($sql);
    }

    return $stmt->fetchAll(PDO::FETCH_ASSOC);
}

// ─── LOGIQUE PRINCIPALE ───────────────────────────────────────────────────────

$erreur    = '';
$clients   = [];
$recherche = '';

// Récupération et assainissement de la saisie utilisateur
if (isset($_GET['q'])) {
    $recherche = trim(strip_tags($_GET['q']));
    // On limite la longueur pour éviter les abus
    $recherche = mb_substr($recherche, 0, 100);
}

try {
    $pdo     = connecterDB();
    $clients = rechercherClients($pdo, $recherche);
} catch (PDOException $e) {
    // Ne jamais afficher les détails d'erreur DB aux clients WAN
    $erreur = 'Impossible de joindre le serveur de données. Veuillez réessayer plus tard.';
    // Journalisation interne (côté serveur uniquement)
    error_log('[ERREUR DB] ' . $e->getMessage() . ' — depuis IP: ' . ($_SERVER['REMOTE_ADDR'] ?? 'inconnue'));
}

?>
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Portail Clients — Réseau DMZ/LAN</title>
    <style>
        /* ── RESET & BASE ── */
        *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

        body {
            font-family: 'Segoe UI', system-ui, sans-serif;
            background: #f0f4f8;
            color: #1a2332;
            min-height: 100vh;
        }

        /* ── EN-TÊTE ── */
        header {
            background: linear-gradient(135deg, #0f4c81, #1976d2);
            color: white;
            padding: 1.5rem 2rem;
            box-shadow: 0 2px 8px rgba(0,0,0,.25);
        }

        header h1 {
            font-size: 1.6rem;
            font-weight: 700;
            letter-spacing: .5px;
        }

        header p {
            font-size: .85rem;
            opacity: .75;
            margin-top: .25rem;
        }

        .badge-dmz {
            display: inline-block;
            background: #ff7043;
            color: #fff;
            font-size: .7rem;
            font-weight: 700;
            padding: .2rem .55rem;
            border-radius: 4px;
            letter-spacing: .5px;
            margin-left: .5rem;
            vertical-align: middle;
        }

        /* ── CONTENEUR ── */
        main {
            max-width: 1100px;
            margin: 2rem auto;
            padding: 0 1rem;
        }

        /* ── FORMULAIRE DE RECHERCHE ── */
        .recherche-form {
            display: flex;
            gap: .75rem;
            margin-bottom: 1.5rem;
            flex-wrap: wrap;
        }

        .recherche-form input[type="search"] {
            flex: 1;
            min-width: 220px;
            padding: .65rem 1rem;
            border: 2px solid #c8d8ea;
            border-radius: 8px;
            font-size: 1rem;
            outline: none;
            transition: border-color .2s;
        }

        .recherche-form input[type="search"]:focus {
            border-color: #1976d2;
        }

        .recherche-form button {
            padding: .65rem 1.4rem;
            background: #1976d2;
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 1rem;
            cursor: pointer;
            font-weight: 600;
            transition: background .2s;
        }

        .recherche-form button:hover { background: #0f4c81; }

        .btn-reset {
            padding: .65rem 1rem;
            background: #ecf0f4;
            color: #555;
            border: 2px solid #c8d8ea;
            border-radius: 8px;
            font-size: .9rem;
            cursor: pointer;
            text-decoration: none;
            display: inline-flex;
            align-items: center;
        }

        /* ── MESSAGES ── */
        .alerte {
            padding: 1rem 1.25rem;
            border-radius: 8px;
            margin-bottom: 1.25rem;
            font-size: .95rem;
        }

        .alerte-erreur  { background: #fde8e8; color: #c0392b; border-left: 4px solid #e74c3c; }
        .alerte-vide    { background: #e8f4fd; color: #2471a3; border-left: 4px solid #2980b9; }
        .alerte-succes  { background: #e8f8f5; color: #1e8449; border-left: 4px solid #27ae60; }

        /* ── TABLEAU ── */
        .carte {
            background: white;
            border-radius: 12px;
            box-shadow: 0 2px 12px rgba(0,0,0,.08);
            overflow: hidden;
        }

        .carte-entete {
            padding: 1rem 1.5rem;
            border-bottom: 1px solid #e8edf2;
            display: flex;
            align-items: center;
            justify-content: space-between;
            flex-wrap: wrap;
            gap: .5rem;
        }

        .carte-entete h2 {
            font-size: 1.05rem;
            color: #2c3e50;
        }

        .compteur {
            font-size: .85rem;
            color: #7f8c8d;
        }

        .table-wrapper { overflow-x: auto; }

        table {
            width: 100%;
            border-collapse: collapse;
            font-size: .9rem;
        }

        thead th {
            background: #f7f9fc;
            color: #555f6e;
            font-weight: 600;
            text-align: left;
            padding: .85rem 1rem;
            border-bottom: 2px solid #e4eaf0;
            white-space: nowrap;
        }

        tbody tr {
            border-bottom: 1px solid #edf1f5;
            transition: background .15s;
        }

        tbody tr:last-child { border-bottom: none; }
        tbody tr:hover { background: #f5f8fc; }

        tbody td {
            padding: .8rem 1rem;
            color: #2c3e50;
            vertical-align: middle;
        }

        .td-id {
            font-weight: 700;
            color: #1976d2;
            width: 60px;
        }

        .td-email a {
            color: #1976d2;
            text-decoration: none;
        }

        .td-email a:hover { text-decoration: underline; }

        .td-date {
            color: #7f8c8d;
            font-size: .82rem;
            white-space: nowrap;
        }

        /* ── PIED DE PAGE ── */
        footer {
            text-align: center;
            padding: 2rem 1rem;
            color: #a0aab4;
            font-size: .8rem;
        }

        footer strong { color: #7f8c8d; }
    </style>
</head>
<body>

<!-- EN-TÊTE -->
<header>
    <h1>Portail Clients <span class="badge-dmz">DMZ</span></h1>
    <p>Données hébergées sur le serveur LAN · Accès sécurisé via pfSense</p>
</header>

<main>

    <!-- FORMULAIRE DE RECHERCHE -->
    <form method="GET" action="" class="recherche-form">
        <input
            type="search"
            name="q"
            value="<?= h($recherche) ?>"
            placeholder="Rechercher par nom ou e-mail…"
            maxlength="100"
            autocomplete="off"
        >
        <button type="submit">🔍 Rechercher</button>
        <?php if ($recherche !== ''): ?>
            <a href="?" class="btn-reset">✕ Réinitialiser</a>
        <?php endif; ?>
    </form>

    <!-- MESSAGE D'ERREUR -->
    <?php if ($erreur !== ''): ?>
        <div class="alerte alerte-erreur">
            ⚠️ <?= h($erreur) ?>
        </div>
    <?php endif; ?>

    <!-- TABLEAU DE RÉSULTATS -->
    <?php if ($erreur === ''): ?>
        <div class="carte">
            <div class="carte-entete">
                <h2>
                    <?php if ($recherche !== ''): ?>
                        Résultats pour « <?= h($recherche) ?> »
                    <?php else: ?>
                        Liste des clients
                    <?php endif; ?>
                </h2>
                <span class="compteur">
                    <?= count($clients) ?> enregistrement(s)
                    <?= count($clients) >= LIMITE_RESULTATS ? '(limite atteinte)' : '' ?>
                </span>
            </div>

            <?php if (count($clients) === 0): ?>
                <div class="alerte alerte-vide" style="margin:1rem 1.5rem">
                    ℹ️ Aucun client trouvé
                    <?= $recherche !== '' ? 'pour cette recherche.' : 'dans la base de données.' ?>
                </div>
            <?php else: ?>
                <div class="table-wrapper">
                    <table>
                        <thead>
                            <tr>
                                <th>#</th>
                                <th>Prénom</th>
                                <th>Nom</th>
                                <th>E-mail</th>
                                <th>Téléphone</th>
                                <th>Adresse</th>
                                <th>Inscription</th>
                            </tr>
                        </thead>
                         <tbody>
                            <?php foreach ($clients as $client): ?>
                                <tr>
                                    <td class="td-id"><?= h($client['id']) ?></td>
                                    <td><?= h($client['nom']) ?></td>
                                    <td>-</td>
                                    <td class="td-email">Non renseigné</td>
                                    <td>-</td>
                                    <td>-</td>
                                    <td class="td-date">
                                        <?= h($client['conservation']) ?> jours
                                    </td>
                                </tr>
                            <?php endforeach; ?>
                        </tbody>
                    </table>
                </div>
            <?php endif; ?>
        </div>
    <?php endif; ?>

</main>

<footer>
    <strong>Architecture réseau :</strong>
    WAN → pfSense → DMZ (<?= h($_SERVER['SERVER_ADDR'] ?? 'Web') ?>)
    → pfSense → LAN (<?= h(DB_HOST) ?>:<?= h(DB_PORT) ?>)
</footer>

</body>
</html>

```

On va mtn rajouter une ligne au init.php sinon notre docker web ne va pas pouvoir recupérer les info de la bdd dans notre interface graphique. On ne fait QUE rajouter, on ne modifie rien de déjà existant 

```bash
meow@debserv:~/projet_cyber$ nano init.sql
```
 
On rajoute ça après les commandes CREATE déjà existante :
```bash

CREATE USER IF NOT EXISTS 'user_web'@'172.%.%.%' IDENTIFIED BY 'PassWeb123!';
GRANT SELECT ON backup_infra_db.* TO 'user_web'@'172.%.%.%';

```
Mtn on va faire un docker compose up mais on a pas de co pour le faire donc on va temporairement changer la route par défaut juste pour telecharger avant de la remettre comme avant. 

```bash

sudo ip link set enp0s8 up
sudo ip route del default
sudo dhcpcd enp0s8
ping -c 3 8.8.8.8
docker compose up -d --build web-interface
```


Mtn on va remettre tous jolie comme avant 

```bash
sudo ip route del default
sudo ip route add default via 192.168.34.254 dev enp0s3
sudo ip link set enp0s8 down
```

Nous pouvons a présent se co sur notre navigateur de notre pc a IP SSH:8080 (perso c'étais 10.37.37.10 mais c'est parceque j'etais en host only chez moi)
On aura avec ça une petite interface graphique toute sympa :):)