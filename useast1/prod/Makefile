BUILD_SITE := "useast1"
BUILD_ENV  := "prod"
FROM       := "manifests/live.yml"
TO         := "manifests/manifest.yml"
SPLIT_JOBS := "runner_z1 runner_z2"

default: split

diff: manifest
	spiff diff $(FROM) $(TO)

split:
	@../../bin/split "$(BUILD_SITE)" "$(BUILD_ENV)" "$(TO)" "$(SPLIT_JOBS)"

help:
	cat ../../bin/help

.PHONY: default diff split help
