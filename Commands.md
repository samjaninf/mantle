# Mantle Commands Support

## Configuration

### .env files


```shell
# Run "$@" in a subshell with service-specific .env loaded
with-env() (
    local env_file="deploy/$1.env"; [[ ! -f "$env_file" ]] || export-env "$env_file"
    "${@:2}"
)

map-env() {
    if have-services '==1'; then
        with-env "$DOCO_SERVICES" "$@"
    else
        foreach-service map-env "$@"
    fi
}
```

### URL Components

The `parse-url` function parses an absolute URL in `$1` and sets `REPLY` to an array containing:

* The URL normalized to include a trailing `/`
* The URL scheme
* The URL host
* The URL port (or an empty string)
* The URL path (or an empty string if no non-`/` path was included)

A variable name can be passed in `$2`: if given, its value is set to the normalized URL and `export`ed.

```shell
parse-url() {
    [[ $1 =~ ([^:]+)://([^/:]+)(:[0-9]+)?(/.*)$ ]] || loco_error "Invalid ${2-URL}: $1"
    REPLY=("${BASH_REMATCH%/}/" "${BASH_REMATCH[1]}" "${BASH_REMATCH[2]}" "${BASH_REMATCH[3]#:}"  "${BASH_REMATCH[4]%/}")
    [[ ! "${2-}" ]] || export "$2=$REPLY"
}

```

## Service Commands

Normally, `up`, `start`, `stop`, etc. apply to all services by default.  But we only want them to apply to development:

```shell
for REPLY in create kill logs pause pull push restart rm scale start stop unpause up; do eval "
doco.$REPLY() {
    if have-services; then
        loco_exec $REPLY \"\$@\"
    else
        with-service dev loco_exec $REPLY \"\$@\"
    fi
}"
done
```

## Developer Commands

These commands execute their arguments inside exactly one service container, as the developer user.  If no services are specified, the default is to run against the `dev` container.

### asdev

Run the specified command line inside the container using the `as-developer` tool.

```shell
doco.asdev() {
    set -- as-developer "$@";
    [[ -t 1 ]] || set -- -T "$@";
    doco cmd "1 asdev" exec "$@";
}
```

### wp

```shell
doco.wp() { doco cmd "1 wp" asdev env PAGER='less' LESS=R wp "$@"; }
```

### db

```shell
doco.db() { doco cmd "1 db" asdev wp db "$@"; }
```

### composer

```shell
doco.composer() { doco cmd "1 composer" asdev composer "$@"; }
```

### imposer

```shell
doco.imposer() { doco cmd "1 imposer" asdev imposer "$@"; }
```

## Other Commands

### dba

The `dba` command executes a `wp db` command in a single container, as the database root user (using user `root` and password `$DB_ROOTPASS`).  If there is an `.env` file for the relevant service, it is loaded so that the service can override the default root user settings (e.g. if it uses a different server).

`dba` subcommands can be defined via `dba.X` functions; if no such function exists, the subcommand is assumed to be a `wp db` subcommand and is passed through as such.  If no dba subcommand is given, a command line interface is started, targeting the `mysql` (meta-)database rather than the container's database.

```shell
doco.dba() {
    if have-services '==1'; then map-env dba "$@"; else doco cmd 1 dba "$@"; fi
}

dba() { if fn-exists "dba.${1-}"; then "dba.${1-}" "${@:2}"; else run-dba; fi; }

run-dba() {
    mysql -h "$DB_HOST" -u root -p "$DB_ROOTPASS" mysql
}

sql-escape() { set -- "${@//\\/\\\\}"; set -- "${@//\'/\\\'}"; REPLY=("$@"); }
```

#### dba mkuser

Creates the database user for the current container, using its `DB_USER` and `DB_PASSWORD`, granting that user access to `DB_NAME`.

```shell
dba.mkuser() {
    sql-escape "$DB_USER" "$DB_PASSWORD"
    printf \
        "GRANT ALL PRIVILEGES ON \`%s\`.* TO '%s'@'%%' IDENTIFIED BY '%s'; FLUSH PRIVILEGES;" \
        "$DB_NAME" "$REPLY" "${REPLY[1]}" | dba
}
```

#### dba dropuser

Drops the `DB_USER` for the current container.

```shell
dba.dropuser() {
    sql-escape "$DB_USER"; printf "DROP USER '%s'@'%%'; FLUSH PRIVILEGES;" "$REPLY" | dba
}
```
