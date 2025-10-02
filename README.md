# WordPress Theme and Plugin JS Internationalization (i18n) Guide 2025
This guide explains how to fully internationalize JavaScript (and PHP) files in a WordPress theme.
WordPress versions: 6.8.1, 6.8.2, 6.8.3

## Bash commands recap - Follow the Step-by-Step Guide for the details

```bash
npm run start 
wp i18n make-pot ./ languages/my-text-domain.pot --exclude="node_modules/*,src/*,vendor/*"
cp languages/my-text-domain.pot languages/{my-text-domain}-{language-code}.po
wp i18n update-po languages/my-text-domain.pot languages/{my-text-domain}-{language-code}.po
wp i18n make-mo languages/{my-text-domain}-{language-code}.po
wp i18n make-json ./languages
npm run build
```

## Step-by-Step Guide

### 1. Prerequisites

Before proceeding, make sure you have:

- **WP-CLI** installed and configured
  - If not installed, follow the instructions at: [https://wp-cli.org/](https://wp-cli.org/)

- **Sufficient PHP Memory**
  - You may need to increase the PHP memory limit to 256M or more if you encounter memory issues
  - To find your php.ini file:
    ```bash
    php --ini
    ```
  - Open the loaded php.ini, look for `memory_limit` and change it to:
    ```
    memory_limit = 512M
    ```
  - Alternatively, add these lines to your wp-config.php file (remember to remove them once you've completed the translation process):
    ```php
    define('WP_MEMORY_LIMIT', '512M');
    define('WP_MAX_MEMORY_LIMIT', '512M');
    ```


### 2. Correctly use wp.i18n methods in your JavaScript files

Use wp.i18n methods directly in your components like this:

```jsx
export function MyComponent() {
    const { __ } =  wp.i18n;
    return(
        <div>
            <p>{__( 'My string', 'my-text-domain' )}</p>
        </div>
    );
}
```

> Do not import @wordpress/i18n methods at the top of your files like this, it currently won't work:
> ```jsx
> import { __ } from '@wordpress/i18n';
> ```

If you previously used the import syntax, you'll need to refactor all your components to use the direct wp.i18n approach shown above. This may be time-consuming but is necessary for proper internationalization.

### 3. Build your theme

For internationalization, you must use the compiled JavaScript files from your build or dist directory, not the original source files.  
Internationalization (i18n) strings are included in compiled .map files. To work with translations:  
- Set your environment to development mode.  
- Keep the .map files while translating.  
- Once translations are complete, delete them before deploying to production.  
- If .map files are missing, ensure your environment is set to development mode and run your development build command before proceeding.

Open a terminal at the root of your theme to build your project:

```bash
npm run start 
# or 
yarn start
# or any other start command present in your package.json
```

### 4. Generate POT file for the theme or the plugin

Create a POT file (translation template):

```bash
wp i18n make-pot ./ languages/my-text-domain.pot --exclude="node_modules/*,src/*,vendor/*"
```

This excludes node_modules, src and php vendor directories, add any other directories you don't want to be scanned for translation.
If your source files are located in a directory other than "src" (such as "source", "js-src", or "assets/js"), modify the exclude pattern accordingly:

```bash
wp i18n make-pot ./ languages/my-text-domain.pot --exclude="node_modules/*,your-source-dir/*,other-dir-to-exclude/*"
```

### 5. Create or update the PO file (translation language)

Respect this filename format for translation language files: `{my-text-domain}-{language-code}.po`
Then, proceed based on your scenario:

**Case 1: The PO file doesn’t exist** 

Create the PO file by copying and renaming the .pot file, then update translations:
```bash
cp languages/my-text-domain.pot languages/{my-text-domain}-{language-code}.po
```

**Case 2: The PO file already exists** 

Update the existing PO file with the latest strings from the .pot file:

```bash
wp i18n update-po languages/my-text-domain.pot languages/{my-text-domain}-{language-code}.po
```

**Case 3: Using Poedit application for translation (PHP only)**

If you choose to translate with Poedit, any subsequent use of `wp i18n update-po` will overwrite the translations you added in Poedit, requiring you to start over. 
Note that Poedit does not automatically scan JavaScript files, so Poedit translation is only suitable for PHP files. 
To manually create the PO file in Poedit:
  - Opening Poedit → New Translation from POT File
  - Selecting my-text-domain.pot and saving as {my-text-domain}-{language-code}.po

### 6. Translate the PO File (translation language)
Translate the resulting PO file in you code editor.

### 7. Generate MO and JSON files

#### Generate the MO file (Compiled translation language)
Create it with WP-CLI in your working directory:
```bash
wp i18n make-mo languages/{my-text-domain}-{language-code}.po
```

#### Generate the JSON files
After your MO file has been properly created, you can proceed with the `wp i18n make-json` command. This command uses the MO file as its source:
```bash
wp i18n make-json ./languages
```

#### Additional i18n commands
For a comprehensive list of available internationalization options and commands, use the WP-CLI help:
```bash
wp i18n --help
```

### You have completed the CLI part. Now you need to load the resulting files into your theme or plugin.
### 8. Load translations in your theme or your plugin

For a theme, use the following PHP function to load the translations:

```php
add_action( 'wp_enqueue_scripts', function() {

    $file = get_template_directory_uri() . '/dist/index.js';
    if(false === is_readable($file)) {
        return;
    }
    $version = filemtime($file);

    wp_enqueue_script( 'my-script', $file, array('wp-element', 'wp-components', 'wp-i18n'), $version, array( 'in_footer' => true ) );
            
    if(defined('WP_LANG_DIR')) {
        wp_set_script_translations( 'my-script', 'my-text-domain', WP_LANG_DIR . '/themes' );
    } else {
        wp_set_script_translations( 'my-script', 'my-text-domain', get_template_directory() . '/languages' );
    }
});
```

For a plugin, use the following PHP function to load the translations:

```php
add_action('wp_enqueue_scripts', function () {
    
	$file = plugins_url('build/index.js', __FILE__);
	if(false === is_readable($file)) {
		return;
	}

	$version = filemtime($file);
	wp_enqueue_script('my-script', $file, array('wp-element', 'wp-components', 'wp-i18n'), $version, array( 'in_footer' => true ) );

	if(defined('WP_LANG_DIR')) {
	    wp_set_script_translations( 'my-script', 'my-text-domain', WP_LANG_DIR . '/plugins' );
	} else {
	    wp_set_script_translations( 'my-script', 'my-text-domain', plugin_basename( dirname( __FILE__ ) ) . '/languages' );
	}
});
```

> **Note:** After several tests, the script has to be enqueued first. Registering, setting translations and finally enqueuing it did not work in my environment. We also need to test the language directory for latest WordPress versions.

### 9. Copy language files to WordPress languages directory

You may need to copy the PO, MO, and JSON files to the wp-content/languages/themes directory of WordPress for the latest WordPress versions.

Here's a function to do that on translation file update, you can adjust the ```$theme_lang_dir = realpath( get_template_directory() . '/languages' );``` part to your needs :

```php
function may_copy_theme_lang_to_languages_dir(): void {
    
    if (false === is_admin() || false === defined('WP_LANG_DIR')) {
        return;
    }

    $theme_lang_dir = realpath( get_template_directory() . '/languages' );

    try{
        if ( false === is_readable( $theme_lang_dir ) ) {
            error_log( 'Theme languages directory does not exist or is not readable: ' . $theme_lang_dir );
            return;
        }
        $extensions = array('mo', 'po', 'json');
        $source_lang_files = array();
        foreach ( $extensions as $ext ) {
            $files = glob( $theme_lang_dir . '/*.' . $ext );
            if ( is_array( $files ) && ! empty( $files ) ) {
                $source_lang_files = array_merge( $source_lang_files, $files );
            }
        }
        if ( empty( $source_lang_files ) ) {
            error_log( 'No language files found in theme languages directory: ' . $theme_lang_dir );
            return;
        }

        $target_lang_dir = WP_LANG_DIR . '/themes';
        if ( is_writable( $target_lang_dir ) ) {
            foreach ( $source_lang_files as $file ) {
                $filename = basename( $file );
                $target_file = $target_lang_dir . '/' . $filename;
                if ( false === is_readable( $target_file ) || filemtime( $file ) > filemtime( $target_file ) ) {
                    if ( copy( $file, $target_file ) ) {
                        error_log( 'Copied language file: ' . $filename );
                    } else {
                        error_log( 'Failed to copy language file: ' . $filename );
                    }
                }
            }
        } else {
            error_log( 'Target languages directory does not exist: ' . $target_lang_dir );
        }
    } catch (\Exception $e) {
        error_log( 'Error copying theme language files: ' . $e->getMessage() );
    }    
}
```
