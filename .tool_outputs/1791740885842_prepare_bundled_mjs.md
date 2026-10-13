// Copies the bundled Valkyrie Client into src-tauri/resources/bundled and verifies its SHA-256.
// The version and checksum come from crates/valkyrie-core/src/constants.rs (single source of truth).
//
//   npm run prepare:bundled                    verify + copy
//   npm run prepare:bundled -- --accept-sha    trust the found jar and write its SHA-256 into constants.rs
//
// Env: VALKYRIE_CLIENT_JAR=<jar or folder>, VALKYRIE_ACCEPT_SHA=1, VALKYRIE_ALLOW_MISSING_CLIENT=1.
import { createHash } from 'node:crypto';
import { copyFileSync, existsSync, mkdirSync, readFileSync, statSync, writeFileSync } from 'node:fs';
import { homedir } from 'node:os';
import { dirname, join, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';

const root = resolve(dirname(fileURLToPath(import.meta.url)), '..');
const constantsPath = join(root, 'crates/valkyrie-core/src/constants.rs');
const constants = readFileSync(constantsPath, 'utf8');
const constant = (name) => {
  const match = constants.match(new RegExp(`${name}: &str = "([^"]+)"`));
  if (!match) throw new Error(`${name} not found in constants.rs`);
  return match[1];
};
const version = constant('BUNDLED_CLIENT_VERSION');
const expected = constant('BUNDLED_CLIENT_SHA256');
const fileName = constant('BUNDLED_CLIENT_FILE');
const jarName = `valkyrieclient-${version}.jar`;
const target = join(root, 'src-tauri/resources/bundled', fileName);
const sha256 = (file) => createHash('sha256').update(readFileSync(file)).digest('hex');
const acceptSha = process.argv.includes('--accept-sha') || process.env.VALKYRIE_ACCEPT_SHA === '1';

if (existsSync(target) && sha256(target) === expected) {
  console.log(`Bundled Valkyrie ${version} is ready.`);
  process.exit(0);
}

/** `set VALKYRIE_CLIENT_JAR="C:\path\x.jar"` in cmd keeps the quotes in the value. */
function cleanPath(value) {
  if (!value) return undefined;
  const trimmed = value.trim().replace(/^["']+|["']+$/g, '');
  if (!trimmed) return undefined;
  try {
    if (existsSync(trimmed) && statSync(trimmed).isDirectory()) {
      const inside = [jarName, fileName].map((name) => join(trimmed, name));
      return inside.find((file) => existsSync(file)) ?? inside[0];
    }
  } catch {
    // unreadable path: report it as "not found" below
  }
  return trimmed;
}

const appData = process.env.APPDATA || join(homedir(), 'AppData', 'Roaming');
const candidates = [
  cleanPath(process.env.VALKYRIE_CLIENT_JAR),
  join(root, `../release/${jarName}`),
  join(root, `release/${jarName}`),
  join(root, `../build/libs/${jarName}`),
  // The copy installed by Launcher 1.x/2.x itself. Its hash is checked like any other.
  join(appData, 'ValkyrieLauncher', 'game', 'mods', jarName),
].filter(Boolean);
const source = candidates.find((file) => existsSync(file));
if (!source) {
  const message = `Bundled Valkyrie ${version} not found. Looked in:\n  ${candidates.join('\n  ')}\n` +
    `Build the client (.\\gradlew.bat build) and copy build/libs/${jarName} into release/, ` +
    'or point VALKYRIE_CLIENT_JAR at the jar (without quotes).';
  if (process.env.VALKYRIE_ALLOW_MISSING_CLIENT === '1') {
    console.warn(`warning: ${message}\nContinuing without it (VALKYRIE_ALLOW_MISSING_CLIENT=1).`);
    process.exit(0);
  }
  console.error(message);
  process.exit(1);
}

const actual = sha256(source);
if (actual !== expected) {
  if (!acceptSha) {
    console.error(
      `SHA-256 mismatch for ${source}\n  expected ${expected}\n  actual   ${actual}\n` +
      'If this jar is the build you want to ship, accept it once:\n' +
      '  npm run prepare:bundled -- --accept-sha\n' +
      '(this writes the new hash into crates/valkyrie-core/src/constants.rs; commit it).',
    );
    process.exit(1);
  }
  const updated = constants.replace(
    /(BUNDLED_CLIENT_SHA256: &str = ")[0-9a-fA-F]+(")/,
    `$1${actual}$2`,
  );
  writeFileSync(constantsPath, updated);
  console.warn(`BUNDLED_CLIENT_SHA256 updated in constants.rs:\n  ${expected}\n  -> ${actual}`);
}
mkdirSync(dirname(target), { recursive: true });
copyFileSync(source, target);
console.log(`Bundled Valkyrie ${version} copied from ${source}.`);
