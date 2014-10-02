---
title: Generating PostgreSQL Database Diagrams with postgresql_autodoc
---
The following series of commands will allow you to generate a png file with a database diagram. This post assumes you are using Ubuntu Linux 10.10 or a derivative distribution. If you are not, you may need to adjust the installation command and name of the package for your distribution.

1. Install postgresql_autodoc: `sudo apt-get install postgresql_autodoc`
2. Ensure your PostgreSQL server is running.
3. Run `postgresql_autodoc -U <username> -d <database> --password=<password> -t dot` substituting appropriate values for `<username>`, `<database>`, and `<password>`.
4. The above will produce a dot file with the same name as the database.
5. Run `dot <database>.dot -Tpng -o <database>.png` substituting your database's name for `<database>`. If you do not want a png file, you can run `dot <database>.dot -Txxx` to get a list of possible output formats.
6. Finally, run `rm <database>.dot` replacing `<database>` with your database's name.
