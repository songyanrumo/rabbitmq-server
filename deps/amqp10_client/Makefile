PROJECT = amqp10_client
PROJECT_DESCRIPTION = AMQP 1.0 client from the RabbitMQ Project
PROJECT_MOD = amqp10_client_app

define PROJECT_APP_EXTRA_KEYS
%% Hex.pm package informations.
	{licenses, ["MPL 2.0"]},
	{links, [
	    {"Website", "https://www.rabbitmq.com/"},
	    {"GitHub", "https://github.com/rabbitmq/rabbitmq-server/deps/amqp10_client"}
	  ]},
	{build_tools, ["make", "rebar3"]},
	{files, [
	    $(RABBITMQ_HEXPM_DEFAULT_FILES)
	  ]}
endef

# Release artifacts are put in $(PACKAGES_DIR).
PACKAGES_DIR ?= $(abspath PACKAGES)

BUILD_DEPS = rabbit_common elvis_mk
DEPS = amqp10_common
TEST_DEPS = rabbit rabbitmq_amqp1_0 rabbitmq_ct_helpers
LOCAL_DEPS = ssl inets crypto

DEP_EARLY_PLUGINS = rabbit_common/mk/rabbitmq-early-test.mk
DEP_PLUGINS = rabbit_common/mk/rabbitmq-macros.mk \
	      rabbit_common/mk/rabbitmq-build.mk \
	      rabbit_common/mk/rabbitmq-hexpm.mk \
	      rabbit_common/mk/rabbitmq-dist.mk \
	      rabbit_common/mk/rabbitmq-run.mk \
	      rabbit_common/mk/rabbitmq-test.mk \
	      rabbit_common/mk/rabbitmq-tools.mk

DEP_PLUGINS += elvis_mk
dep_elvis_mk = git https://github.com/inaka/elvis.mk.git master

# FIXME: Use erlang.mk patched for RabbitMQ, while waiting for PRs to be
# reviewed and merged.

ERLANG_MK_REPO = https://github.com/rabbitmq/erlang.mk.git
ERLANG_MK_COMMIT = rabbitmq-tmp

include ../../rabbitmq-components.mk
include ../../erlang.mk

# --------------------------------------------------------------------
# Compiler flags.
# --------------------------------------------------------------------

# gen_fsm is deprecated starting from Erlang 20, but we want to support
# Erlang 19 as well.

ERTS_VER := $(shell erl -version 2>&1 | sed -E 's/.* version //')
ERLANG_20_ERTS_VER := 9.0

ifeq ($(call compare_version,$(ERTS_VER),$(ERLANG_20_ERTS_VER),>=),true)
ERLC_OPTS += -Dnowarn_deprecated_gen_fsm
endif

# Dialyze the tests.
DIALYZER_OPTS += --src -r test

# --------------------------------------------------------------------
# ActiveMQ for the testsuite.
# --------------------------------------------------------------------

ACTIVEMQ_VERSION := 5.14.4
ACTIVEMQ_URL := 'https://archive.apache.org/dist/activemq/$(ACTIVEMQ_VERSION)/apache-activemq-$(ACTIVEMQ_VERSION)-bin.tar.gz'

ACTIVEMQ := $(abspath test/system_SUITE_data/apache-activemq-$(ACTIVEMQ_VERSION)/bin/activemq)
export ACTIVEMQ

$(ACTIVEMQ): \
  test/system_SUITE_data/apache-activemq-$(ACTIVEMQ_VERSION)-bin.tar.gz
	$(gen_verbose) cd "$(dir $<)" && tar zxf "$(notdir $<)"

test/system_SUITE_data/apache-activemq-$(ACTIVEMQ_VERSION)-bin.tar.gz:
	$(gen_verbose) $(call core_http_get,$@,$(ACTIVEMQ_URL))

tests:: $(ACTIVEMQ)

ct ct-system: $(ACTIVEMQ)

# --------------------------------------------------------------------
# Distribution.
# --------------------------------------------------------------------

.PHONY: distribution

distribution: docs source-dist package

docs:: edoc
edoc: doc/overview.edoc

doc/overview.edoc: src/overview.edoc.in
	mkdir -p doc
	sed -e 's:%%VERSION%%:$(PROJECT_VERSION):g' < $< > $@

.PHONY: source-dist clean-source-dist

SOURCE_DIST_BASE ?= $(PROJECT)
SOURCE_DIST_SUFFIXES ?= tar.xz zip
SOURCE_DIST ?= $(PACKAGES_DIR)/$(SOURCE_DIST_BASE)-$(PROJECT_VERSION)-src

# The first source distribution file is used by packages: if the archive
# type changes, you must update all packages' Makefile.
SOURCE_DIST_FILES = $(addprefix $(SOURCE_DIST).,$(SOURCE_DIST_SUFFIXES))

.PHONY: $(SOURCE_DIST_FILES)

source-dist: $(SOURCE_DIST_FILES)
	@:

RSYNC ?= rsync
RSYNC_V_0 =
RSYNC_V_1 = -v
RSYNC_V_2 = -v
RSYNC_V = $(RSYNC_V_$(V))
RSYNC_FLAGS += -a $(RSYNC_V)		\
	       --exclude '.sw?' --exclude '.*.sw?'	\
	       --exclude '*.beam'			\
	       --exclude '*.d'				\
	       --exclude '*.pyc'			\
	       --exclude '.git*'			\
	       --exclude '.hg*'				\
	       --exclude '.travis.yml'			\
	       --exclude '.*.plt'			\
	       --exclude '$(notdir $(ERLANG_MK_TMP))'	\
	       --exclude 'cover/'			\
	       --exclude 'deps/'			\
	       --exclude 'ebin/'			\
	       --exclude 'erl_crash.dump'		\
	       --exclude '$(notdir $(DEPS_DIR))/'	\
	       --exclude 'doc/'				\
	       --exclude 'hexer*'			\
	       --exclude 'logs/'			\
	       --exclude 'plugins/'			\
	       --exclude '$(notdir $(DIST_DIR))/'	\
	       --exclude 'test'				\
	       --exclude 'xrefr'			\
	       --exclude '/$(notdir $(PACKAGES_DIR))/'	\
	       --exclude '/PACKAGES/'			\
	       --delete					\
	       --delete-excluded

TAR ?= tar
TAR_V_0 =
TAR_V_1 = -v
TAR_V_2 = -v
TAR_V = $(TAR_V_$(V))

GZIP ?= gzip
BZIP2 ?= bzip2
XZ ?= xz

ZIP ?= zip
ZIP_V_0 = -q
ZIP_V_1 =
ZIP_V_2 =
ZIP_V = $(ZIP_V_$(V))

.PHONY: $(SOURCE_DIST)

$(SOURCE_DIST): $(ERLANG_MK_RECURSIVE_DEPS_LIST)
	$(verbose) mkdir -p $(dir $@)
	$(gen_verbose) $(RSYNC) $(RSYNC_FLAGS) ./ $@/
	$(verbose) echo "$(PROJECT_DESCRIPTION) $(PROJECT_VERSION)" > $@/git-revisions.txt
	$(verbose) echo "$(PROJECT) $$(git rev-parse HEAD) $$(git describe --tags --exact-match 2>/dev/null || git symbolic-ref -q --short HEAD)" >> $@/git-revisions.txt
	$(verbose) for dep in $$(cat $(ERLANG_MK_RECURSIVE_DEPS_LIST) | grep -v '/$(PROJECT)$$' | LC_COLLATE=C sort); do \
		$(RSYNC) $(RSYNC_FLAGS) \
		 $$dep \
		 $@/deps; \
		if test -f $@/deps/$$(basename $$dep)/erlang.mk && \
		   test "$$(wc -l $@/deps/$$(basename $$dep)/erlang.mk | awk '{print $$1;}')" = "1" && \
		   grep -qs -E "^[[:blank:]]*include[[:blank:]]+(erlang\.mk|.*/erlang\.mk)$$" $@/deps/$$(basename $$dep)/erlang.mk; then \
			echo "include ../../erlang.mk" > $@/deps/$$(basename $$dep)/erlang.mk; \
		fi; \
		sed -E -i.bak "s|^[[:blank:]]*include[[:blank:]]+\.\./.*erlang.mk$$|include ../../erlang.mk|" \
		 $@/deps/$$(basename $$dep)/Makefile && \
		rm $@/deps/$$(basename $$dep)/Makefile.bak; \
		(cd $$dep; echo "$$(basename "$$dep") $$(git rev-parse HEAD) $$(git describe --tags --exact-match 2>/dev/null || git symbolic-ref -q --short HEAD)") >> $@/git-revisions.txt; \
	done
	$(verbose) for file in $$(find $@ -name '*.app.src'); do \
		sed -E -i.bak \
		  -e 's/[{]vsn[[:blank:]]*,[[:blank:]]*(""|"0.0.0")[[:blank:]]*}/{vsn, "$(PROJECT_VERSION)"}/' \
		  $$file; \
		rm $$file.bak; \
	done
	$(verbose) rm $@/README.in
	$(verbose) cp README.in $@/README
	$(verbose) if test "$(BUILD_DOC)"; then cat "$(BUILD_DOC)" >> $@/README; fi

# TODO: Fix file timestamps to have reproducible source archives.
# $(verbose) find $@ -not -name 'git-revisions.txt' -print0 | xargs -0 touch -r $@/git-revisions.txt

$(SOURCE_DIST).tar.gz: $(SOURCE_DIST)
	$(gen_verbose) cd $(dir $(SOURCE_DIST)) && \
		find $(notdir $(SOURCE_DIST)) -print0 | LC_COLLATE=C sort -z | \
		xargs -0 $(TAR) $(TAR_V) --no-recursion -cf - | \
		$(GZIP) --best > $@

$(SOURCE_DIST).tar.bz2: $(SOURCE_DIST)
	$(gen_verbose) cd $(dir $(SOURCE_DIST)) && \
		find $(notdir $(SOURCE_DIST)) -print0 | LC_COLLATE=C sort -z | \
		xargs -0 $(TAR) $(TAR_V) --no-recursion -cf - | \
		$(BZIP2) > $@

$(SOURCE_DIST).tar.xz: $(SOURCE_DIST)
	$(gen_verbose) cd $(dir $(SOURCE_DIST)) && \
		find $(notdir $(SOURCE_DIST)) -print0 | LC_COLLATE=C sort -z | \
		xargs -0 $(TAR) $(TAR_V) --no-recursion -cf - | \
		$(XZ) > $@

$(SOURCE_DIST).zip: $(SOURCE_DIST)
	$(verbose) rm -f $@
	$(gen_verbose) cd $(dir $(SOURCE_DIST)) && \
		find $(notdir $(SOURCE_DIST)) -print0 | LC_COLLATE=C sort -z | \
		xargs -0 $(ZIP) $(ZIP_V) $@

clean:: clean-source-dist

clean-source-dist:
	$(gen_verbose) rm -rf -- $(SOURCE_DIST_BASE)-*

package: dist
	cp $(DIST_DIR)/*.ez $(PACKAGES_DIR)
