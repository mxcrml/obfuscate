# Tutoriel pour obsfusquer les liens des mégas menus sur Wordpress (Version BETA)

Voici les codes générés avec GPT-4 en enseignant le méthode de Fred Bobet (pour les sites E-Commerce en 2021). Il faut bien sûr adapter le code en fonction du site. Pour faciliter ce processus, j'ai demandé à GPT-4 de nous faire un filtre afin qu'on puisse appliquer les techniques d'obsfuscations que sur certaines catégories lorsque cela était pertinent, il s'agit de la variable  *$allowed_categories*

## Logique d'obfuscation des liens

Nous créons une fonction PHP (à insérer dans le fichier theme functions.php) *fonction obfuscate_link()*, qui définit les critères d'obfuscation des liens en fonction des conditions suivantes [a + (b1 ou b2)] :

- (a) Le premier segment (page mère) des URL est le même que la page actuelle.
- (b1) Le lien a exactement deux segments de plus que l'URL actuelle
- (b2) Si l'URL actuelle a au moins trois segments (page petite-fille) et que l'avant-dernier segment (page fille) du lien n'est pas identique à l'avant-dernier segment (page fille) de l'URL actuelle, alors il faut obfusquer le lien.

## Fonction obfuscate à placer dans le fichier functions.php
```
function obfuscate_link($url, $text, $current_url, $allowed_categories = []) {
  $url_path = parse_url($url, PHP_URL_PATH);
  $current_url_path = parse_url($current_url, PHP_URL_PATH);

  $url_segments = explode('/', trim($url_path, '/'));
  $current_url_segments = explode('/', trim($current_url_path, '/'));

  // Si aucune catégorie autorisée n'est spécifiée ou si la catégorie actuelle est autorisée
  $category_allowed = empty($allowed_categories) || in_array($current_url_segments[0], $allowed_categories);

  // Vérifie si l'URL répond aux critères d'obfuscation
  $obfuscate = false;

  if ($category_allowed) {
    if ($url_segments[0] === $current_url_segments[0] && count($url_segments) === count($current_url_segments) + 2) {
      $obfuscate = true;
    } elseif (count($current_url_segments) >= 3 && $url_segments[count($url_segments) - 2] !== $current_url_segments[count($current_url_segments) - 2]) {
      $obfuscate = true;
    }
  }

  if ($obfuscate) {
    // Génère un bouton obfusqué
    return "<button onclick=\"location.href='{$url}'\">{$text}</button>";
  } else {
    // Génère un lien standard
    return "<a href=\"{$url}\">{$text}</a>";
  }
}
```
## Filtre à insérer en dessous de la fonction obsfucate

**Bien penser à éditer la variable *$allowed_categories = ["cat1", "cat2"];* cat1 et cat2 étant deux exemples de pages mères / catégories.**
```
function filter_wp_nav_menu_objects($sorted_menu_items) {
  $current_url = (isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? "https" : "http") . "://$_SERVER[HTTP_HOST]$_SERVER[REQUEST_URI]";
  $allowed_categories = ["cat1", "cat2"]; // Les catégories autorisées

  foreach ($sorted_menu_items as $item) {
    $item->url = obfuscate_link($current_url, $allowed_categories);
    $item->title = ''; // Supprime le titre de l'élément du menu, car il sera inclus dans le lien ou le bouton
  }

  return $sorted_menu_items;
}

add_filter('wp_nav_menu_objects', 'filter_wp_nav_menu_objects', 10, 1);
```

### Notes importantes : 
La fonction filter_wp_nav_menu_objects utilisée dans cet exemple est conçue pour filtrer les objets de menu générés par la fonction wp_nav_menu() de WordPress. Cependant, cette fonction ne prend pas nécessairement en compte les méga menus.

Les méga menus sont généralement créés à l'aide de plugins supplémentaires ou de thèmes personnalisés qui modifient le comportement par défaut de wp_nav_menu(). Pour que la fonction filter_wp_nav_menu_objects fonctionne avec les méga menus, vous devrez peut-être adapter la fonction en fonction du plugin ou du thème que vous utilisez pour créer les méga menus.

Chaque plugin ou thème de méga menu peut fonctionner différemment, il est donc difficile de fournir une solution universelle pour tous les méga menus. Vous devrez examiner la documentation ou le code du plugin/thème que vous utilisez et adapter la fonction filter_wp_nav_menu_objects en conséquence.

## Généralisation de la fonction obfuscate

Pour créer des règles d'obfuscation sur mesure en fonction des pages, vous pouvez personnaliser la fonction obfuscate_link en ajoutant des conditions spécifiques pour les pages et les liens que vous souhaitez obfusquer. 

Voici une version modifiée de la fonction obfuscate_link qui obfusque les liens vers certaines pages (par exemple, contact, mentions légales) sur toutes les pages, sauf la page d'accueil.

```
function obfuscate_link($url, $text, $current_url) {
  $url_path = parse_url($url, PHP_URL_PATH);
  $current_url_path = parse_url($current_url, PHP_URL_PATH);

  $url_segments = explode('/', trim($url_path, '/'));
  $current_url_segments = explode('/', trim($current_url_path, '/'));

  // Ajoutez les slugs des pages que vous souhaitez obfusquer dans ce tableau
  $pages_to_obfuscate = ['contact', 'mentions-legales'];

  // Vérifie si l'URL actuelle n'est pas la page d'accueil
  $not_homepage = !empty($current_url_segments[0]);

  // Vérifie si le lien correspond à l'une des pages à obfusquer
  $is_obfuscate_page = in_array($url_segments[count($url_segments) - 1], $pages_to_obfuscate);

  // Obfusque le lien si les conditions sont remplies
  if ($not_homepage && $is_obfuscate_page) {
    return "<button onclick=\"location.href='{$url}'\">{$text}</button>";
  } else {
    return "<a href=\"{$url}\">{$text}</a>";
  }
}
```

## Nouvelle version éditée après les commentaires de Fred 

Logique corrigée à partir du schema envoyé + intégration des exceptions

```
function obfuscate_link_custom($url, $text, $current_url) {
    $url_path = parse_url($url, PHP_URL_PATH);
    $current_url_path = parse_url($current_url, PHP_URL_PATH);

    $url_segments = explode('/', trim($url_path, '/'));
    $current_url_segments = explode('/', trim($current_url_path, '/'));

    $current_segment_count = count($current_url_segments);
    $url_segment_count = count($url_segments);

    $obfuscate = false;

    if ($current_segment_count === 1) { // Page d'accueil
        if ($url_segment_count > 2) {
            $obfuscate = true;
        }
    } elseif ($current_segment_count === 2) { // Pages mères
        if ($url_segment_count > 3 && $url_segments[0] === $current_url_segments[0]) {
            $obfuscate = true;
        }
    } elseif ($current_segment_count === 3) { // Pages filles
        if ($url_segment_count > 4 && $url_segments[0] === $current_url_segments[0] && $url_segments[1] === $current_url_segments[1]) {
            $obfuscate = true;
        }
    } elseif ($current_segment_count >= 4) { // Pages petites-filles et suivantes
        if ($url_segment_count >= $current_segment_count + 1 && $url_segments[0] === $current_url_segments[0] && $url_segments[1] === $current_url_segments[1] && $url_segments[2] === $current_url_segments[2]) {
            $obfuscate = true;
        }
    }

    if ($obfuscate) {
        return "<button onclick=\"location.href='{$url}'\">{$text}</button>";
    } else {
        return "<a href=\"{$url}\">{$text}</a>";
    }
}
```
