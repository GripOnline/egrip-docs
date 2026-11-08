# Setting up a new site

[← Developer's guide](../developers-guide.md)

The steps to set up a new e-Grip 4 site. Hosting- and server-specific steps (which development server, how databases
are created there) are internal and not described here.

## Repository

1. Create a new **private** repository in the GripOnline GitHub organization.
2. Name it `egrip-<site>`. Use underscores for hyphens in the site name (for example `egrip-<name>_<suffix>`).
3. Clone it into your development environment, and rename the directory to the site name with hyphens instead of
   underscores:

   ```bash
   git clone git@github.com:GripOnline/egrip-<site>.git
   ```

   ```bash
   mv egrip-<site> <site-with-hyphens>
   ```

4. Copy the files of the `egrip4` reference repository into the new directory (for example by downloading it as a
   ZIP archive and extracting it there).
   The public [egrip-demo](https://github.com/GripOnline/egrip-demo) repository shows what a minimal e-Grip 4 site
   consists of.
5. Update the branch name in `vwm_config.xml` and `composer.json` to the new branch name.
6. Commit these files to the new repository.

## Database

Create a database for your development environment, without a separate database user. Use a name that identifies the
site and the environment, for example `<site>-<environment>`.

## Development environment in vwm_config.xml

1. Make sure the environment name does **not** start with `dev_` (for example `<environment name="<environment>" ...>`).
2. Set the `database` parameter. Note the scheme: `mysqli://`.
3. Commit `vwm_config.xml`. Do not commit credentials to a public location.
