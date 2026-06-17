# Guidelines for Preventing Malicious npm Packages

Malicious packages are surging in public repositories, with npm showing the highest concentration. Industry sources indicate that in 2025, more than 90% of malicious packages detected in the wild were found in the npm ecosystem. Nearly 60% of the open-source libraries used originate from npm.

Incidents involving malicious packages and their potential impact underscore the need for engineers to stay alert and apply secure practices when working with open-source libraries and managing dependencies.

To minimize the risk of malicious packages, the following guidelines define secure practices for using and updating npm libraries.
These practices form the foundation for guardrails that engineering teams should integrate into their pipelines and workflows.

## 1. Clean install in CI/CD pipelines for reproducibility

To ensure reproducible and fast CI/CD builds, commit the generated package-lock.json file and use [npm clean install](https://docs.npmjs.com/cli/commands/npm-ci).

| Don't this in your pipeline | Instead, do this |
|-----------------------------|------------------|
| `npm install`               | `npm ci`         |

`npm install` installs dependencies listed in package.json. It installs missing packages, updates outdated ones, and can add new ones.
When used in a CI/CD pipeline, it tries to use package-lock.json if it exists. But may update it if there are changes or new packages.
This opens up a potential vector for a compromised version of a package to get in to your build system.

On the other hand, `npm ci` removes your entire node_modules and installs only what’s needed based on your lockfile.

While `npm install` is handy for local development, it is not quite suitable for CI/CD. Use `npm ci` instead.

## 2. Ignore scripts

One of the major supply chain risks is that malicious or compromised npm packages can execute arbitrary commands or code during installation.
This has been a well-known attack vector for years and continues to be exploited.

Scripts in npm are commands defined in the scripts section of package.json (like preinstall / postinstall).
These scripts can run automatically at certain points, such as when you install a package.

### 2.1 Ignore scripts during lifecycle events

*ignore-scripts=true* tells npm to not run any scripts during install or other lifecycle events. 
Use it by default for safety, but be aware that some packages may not work correctly without their scripts.

- It is highly recommended to set it in your global configuration to disable running all package lifecycle scripts when it installs, updates, or removes packages.

  `npm config set ignore-scripts true`

- Alternatively, you could disable scripts for your project in `npmrc` file.

  `.npmrc`:

  `ignore-scripts=true`

- The least preferred way is to remember and disable scripts when performing ad-hoc package install.

  `npm install --ignore-scripts <package-name>`

Some packages (like those with native bindings, e.g., bcrypt) require install scripts to compile code.
Such packages may not work correctly until you allow scripts to run.
If you use packages that rely on lifecycle scripts for legitimate reasons, you can use a plugin like [@lavamoat/allow-scripts](https://www.npmjs.com/package/@lavamoat/allow-scripts) to explicitly create an allowlist of packages authorized to run lifecycle scripts.

Here's how the allowlist would look like in the package.json file on a project using the popular image processing package [sharp](https://www.npmjs.com/package/sharp):

```json
{
  "lavamoat": {
    "allowScripts": {
      "sharp": true
    }
  }
}
```

### 2.2 Disable Git-based dependencies

Git dependencies, direct or transitive, can include `.npmrc` files that override the git executable path. This enables arbitrary code execution during install even when using `--ignore-scripts`. 

The new `--allow-git` flag can be used to limit the ability for npm to fetch dependencies from git references. That is, dependencies that point to a git repo instead of a version or semver range.

It is recommended to set npm's allow-git configuration to disable all install scripts.

`npm config set allow-git none`

A less preferred way is to disable npm's post-install scripts when performing ad-hoc package install using the command line:

`npm install --ignore-scripts --allow-git=none <package-name>`

## 3. Automate dependency updates with a 'cooldown'

### 3.1 'Cooldown' in npm
When a malicious package is published on npm, the npm maintainers typically remove it within a day or so at least for popular packages once they become aware of the issue.
Reviews of past malicious package incidents indicate that the impact can often be prevented by introducing a short delay before applying dependency updates.

Therefore:
- Deliberately introducing a reasonable delay when updating dependencies helps reduce the risk of ingesting malicious packages.
- CISO recommends implementing a 72-hour delay before updating dependencies as a simple safeguard against malicious packages, reducing the risk of them entering the codebase while balancing automation with security.

It is preferable to set it globally:

`npm config set min-release-age 3`

Alternatively, set min-release-age in your `.npmrc`:

`min-release-age=3`

The least preferred way is to remember and set it at the time of ad-hoc package install:

`npm install <package-name> --min-release=3`

### 3.2 'Cooldown' in automated dependency updates

Updating the dependency libraries used in your application is essential to keep your code base maintainable and secure over time.
- Directly upgrading all dependencies to their latest versions can expose your application to security vulnerabilities, dependency confusion attacks, and malicious packages released from compromised accounts.
- Tools such as Dependabot and Renovate help you automate and manage the dependency libraries.

Please check out the [engineering documentation](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference#cooldown-) on Dependabot.

- Dependabot offers a [cooldown](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference#cooldown-) option.
- Renovate offers [minimumReleaseAge](https://docs.renovatebot.com/key-concepts/minimum-release-age/) feature.

## 4. Omit dev dependencies in production deployments

Before deployment, ensure your deployments are clean and minimal.

`npm prune --omit=dev`

- `prune` deletes any extra packages in node_modules that aren’t listed in your package.json.

- `--omit=dev` installs only the dependencies needed to run your app and skips devDependencies.

For clean, predictable installs in CI/CD, use:

`npm ci --omit=dev`

It ensures a fresh, minimal, and predictable install, stripping out dev dependencies — ideal for production deployments.

| Command                | When to use it                                                                                                                         |
|------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `npm prune --omit=dev` | Use this if you've already installed everything and just want to strip dev dependencies.                                               |
| `npm ci --omit=dev`    | Clean install of only prod deps (from lockfile). <br/>Use this if you're setting up a production build or deployment in your pipeline. |

This helps to:

- Reduce security risks: Fewer packages means less attack surface.
- Keep your deployment clean: No leftover or unnecessary packages.
- Saves disk space: Only what’s needed is deployed

## 5. Verify signatures of the packages you are using

You can easily verify that the packages you are using have valid registry-provided cryptographic signatures and check their provenance attestations.

The audit signatures command helps to detect if the package has been tampered with.

`npm audit signatures --json --include-attestations`

The `--include-attestations` flag is available starting v11.12.0. It is recommended to always verify signatures during development. 
- If the signature is missing, apply caution and consider if the package is trust worthy and essential for your applicaiton.
- If the verification fails as invalid, investigate the affected package for tampering, and reinstall only from a trusted source and lock to a verified version.

## 6. Security scan: npm audit and Checkmarx

npm audit is a built-in feature of npm that helps identify security vulnerabilities in your project’s dependencies, including malicious packages.
It is a native feature of npm and is useful during development.

Checkmarx is a comprehensive Static Application Security Testing (SAST) and Software Component Analysis (SCA) platform.
Checkmarx offers strong capabilities like SBOM generation, scan for vulnerabilities in code, deep transitive scanning of dependencies for vulnerabilities and malicious packages, licensing violations, triage, reporting and integration with other systems.
It enables Engineering teams, IT Risk teams, Asset Owners, OnePipeline, the CISO, and other stakeholders to centrally establish governance, fulfill regulatory obligations, and maintain compliance with the DORA regulation.

Therefore, Checkmarx is required to be used for SAST and SCA. Code should be scanned using Checkmarx before release, in accordance with the Global Security Testing Strategy.

It is highly recommended to configure the build to fail if there are hits on known malicious packages. Investigate and start the security incident process.

| Use as desired | Mandatory for release     |
|----------------|---------------------------|
| `npm audit`    | CheckmarxOne SAST and SCA |

## 7. Claim a dedicated organisation namespace in npm

An attacker could register their own package on the public npm registry that matched an unclaimed internal package name of an organisation.
It has been proven that this type of dependency confusion exploit can lead to the execution of malicious code.

The key takeaway is that organisations should proactively claim its namespace on npm to mitigate this attack vector. 
Security team must initiate a program to accomplish this in collaboration with organisation’s front-end teams and other stakeholders.

In the future, organization-scoped packages should be adopted (e.g., @org-name/internal-lib) for all libraries developed internally.

---

#### References
- [OWASP npm Security best practices](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html)
- [Verifying ECDSA registry signatures](https://docs.npmjs.com/verifying-registry-signatures)
- [Trusted publishing for npm packages](https://docs.npmjs.com/trusted-publishers)
- [npm: Threats and Mitigations](https://docs.npmjs.com/threats-and-mitigations)
- [Using private packages in a CI/CD workflow](https://docs.npmjs.com/using-private-packages-in-a-ci-cd-workflow)
- [Sonatype State of Software Supply Chain Report](https://www.sonatype.com/state-of-the-software-supply-chain/2026/open-source-malware)
