# Jupyter / IPython disaster recovery

This document summarises the disaster recovery procedure for Jupyter notebooks. While Jupyter has an autosave feature, it is not infallible, and in rare cases manual recovery of significant amounts of work can be necessary. This advice was adapted from [this Medium article](https://medium.com/flatiron-engineering/recovering-from-a-jupyter-disaster-27401677aeeb) &ndash; note that the database schema appears to have changed very marginally from what is described there (specifically the table ```session``` is now called ```sessions```).

1. Jupyter routinely makes automated backups of each notebook. In the first instance, look in the hidden subdirectory ```.ipynb_checkpoints``` (within the working directory of the notebook) for the most recent autosave.
2. In some cases, this might be badly out of date. This has been observed when Firefox is updated while it is being used to run Jupyter &ndash; it quietly suppresses necessary functionality, such as being able to write to the file system.
3. The recovery method of last resort, then, is to find the IPython kernel's database file. This is an SQLite database that contains a record of every instruction issued to the kernel associated with the relevant Jupyter profile. Thus, recovery of all (executed) code cells is possible in principle; note that non-code cells (*e.g.* Markdown), and cell outputs, may not be so recovered.
4. For safety, it is recommended to make a copy of the file ```~/.ipython/profile_default/history.sqlite``` (substituting profile name if necessary). This database may then be interrogated to recover the desired data by looking up the session ID in the ```session``` table and performing a join on the ```history``` table. For example,

    ```{sql}
    SELECT
    s.session, h.line, h.source
    FROM sessions AS s
    INNER JOIN history AS h ON s.session = h.session
    WHERE s.start > '2020-05-10'
    AND h.source LIKE 'def%';
    ```

    will recover all cells beginning with a ```def``` statement executed after the given date. A GUI SQLite frontend tool such as [DB Browser for SQLite](https://sqlitebrowser.org/) is ideal for exploring the tables and manually extracting the  desired code; if not present it may be installed via the package ```sqlitebrowser```  (should work for any package manager). With the ```jupytext``` tool, an appropriately formatted Python script can be automatically converted to a Jupyter notebook file, per the Medium article linked above:

    ```{bash}
    sqlite3 history.sqlite "SELECT '# @@ Cell '|| line || CHAR(10) || source || CHAR(10) FROM history WHERE session = SESSION_ID;" > myoutput.py
    jupytext — to ipynb myoutput.py
```

    once the session ID is known.