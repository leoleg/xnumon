#DEBUG:=	1

ifdef DEBUG
BUILD_INFO+=	DEBUG
#FEATURES+=	-DDEBUG_REFS
#FEATURES+=	-DDEBUG_PROCMON
#FEATURES+=	-DDEBUG_EXECIMAGE
#FEATURES+=	-DDEBUG_EXIT
#FEATURES+=	-DDEBUG_CHDIR
#FEATURES+=	-DDEBUG_PREPQUEUE
#FEATURES+=	-DDEBUG_CACHE
#FEATURES+=	-DDEBUG_AUDITPIPE
endif

PKGNAME:=	xnumon

include Mk/buildinfo.mk

DEVIDAPPL?=	"6BF4C67B70848E7A0635F1F1ADC6C9F869B69CEE"
DEVIDINST?=	"CD9D6DF08F96B0F3EC99A986CB0BF28C68556D4D"
MACOSX_VERSION_MIN=10.12
ifdef DEVIDINST
PRODUCTBUILDFLAGS+=--sign $(DEVIDINST)
endif

include Mk/xcode.mk

CPPFLAGS+=	$(FEATURES)
ifndef DEBUG
CPPFLAGS+=	-DNDEBUG
CFLAGS+=	-O3
LDFLAGS+=	-O3
else
CFLAGS+=	-g
LDFLAGS+=	-g
endif

CFLAGS+=	-arch x86_64
CFLAGS+=	-std=c11 \
		-Wall -Wextra -pedantic
#ifdnef DEBUG
CFLAGS+=	-Wno-unknown-pragmas # open_memstream
CFLAGS+=	-Wno-gnu # minmax.h
#endif
CFLAGS+=	-D_FORTIFY_SOURCE=2 -fstack-protector-all

LDFLAGS+=	-arch x86_64

LIBS+=		-lbsm \
		-framework CoreFoundation \
		-framework Security \
		-framework IOKit

# openssl
#CPPFLAGS+=	-DUSE_OPENSSL
#CFLAGS+=	-I/opt/local/include
#LDFLAGS+=	-L/opt/local/lib
#LIBS+=		-lcrypto

MAINSRCS=	$(shell grep -l '^main\b' *.c)
TARGETS=	$(MAINSRCS:.c=)
SRCS=		$(filter-out $(TARGETS:=.c),$(wildcard *.c))
HDRS=		$(wildcard *.h kext/xnumon.h)
OBJS=		$(SRCS:.c=.o)
MKFS=		$(wildcard Makefile GNUmakefile Mk/*.mk)

EXTSRCS=	tommylist.c tommylist.h tommychain.h tommytypes.h \
		tommyhashdyn.c tommyhashdyn.h \
		tommyhashtbl.c tommyhashtbl.h \
		tommyhash.c tommyhash.h \
		memstream.c memstream.h map.h
GITHUBRAW?=	https://raw.githubusercontent.com
TOMMYTAG?=	v2.2
MAPTAG?=	master

all: $(TARGETS)

$(OBJS) $(TARGETS:=.o): $(MKFS)

build.o: CPPFLAGS+=$(BUILD_CPPFLAGS)
build.o: build.c FORCE

%.o: %.c $(HDRS)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $<

$(TARGETS): %: $(OBJS) %.o
	$(CC) $(LDFLAGS) -o $@ $^ $(LIBS)

sign: $(TARGETS:=.signed)

%.signed: % chkcs
	cp $< $@
	@#strip $@
	$(CODESIGN) --options library -s $(DEVIDAPPL) -i ch.roe.$< -f $@
	$(CODESIGN) -d --verbose=4 $@
	$(CODESIGN) --verify --deep --strict --verbose=4 $@
	./chkcs $@
	@#spctl --verbose=4 --assess --type execute $@

copyright: $(filter-out $(EXTSRCS),$(wildcard *.c *.h)) kext/*.c kext/*.h
	Mk/bin/copyright.py $^
	$(MAKE) -C test $@

fetch: $(EXTSRCS)

map.h:
	curl -L -O $(GITHUBRAW)/swansontec/map-macro/$(MAPTAG)/$@
	xattr -c $@

memstream.h memstream.c:
	curl -L -O http://piumarta.com/software/memstream/memstream-0.1/$@
	xattr -c $@

tommy%.h:
	curl -L -O $(GITHUBRAW)/amadvance/tommyds/$(TOMMYTAG)/tommyds/$@
	xattr -c $@

tommy%.c:
	curl -L -O $(GITHUBRAW)/amadvance/tommyds/$(TOMMYTAG)/tommyds/$@
	xattr -c $@

clean:
	rm -rf $(TARGETS) *.signed *.o *.dSYM

test:
	$(MAKE) -C test $@

kext:
	$(MAKE) -C kext all

kextclean:
	$(MAKE) -C kext clean

travis: all kext
	./xnumon -V

pkg: $(PKGNAME)-$(BUILD_VERSION).pkg

LICENSE_HTML_SRCS:=	pkg/resources/license.header.html \
			LICENSE \
			pkg/resources/license.separator.html \
			LICENSE.third \
			pkg/resources/license.footer.html

pkg/resources/license.html: $(LICENSE_HTML_SRCS) $(MKFS)
	cat $(LICENSE_HTML_SRCS) >$@

# --no-wrap is broken for HTML to RTF conversion in pandoc 1.12.4.2.
# The last sed removes \line keywords not followed by { from the generated RTF.
pkg/resources/readme.rtf: README.md $(MKFS)
	cat $< |\
	sed 's/^.*__BUILD_VERSION__.*$$/~~~§$(PKGNAME) $(BUILD_VERSION) (built $(BUILD_DATE))§~~~/' |\
	tr '§' '\n' |\
	grep -v '\[!\[' |\
	pandoc -f markdown_github -t rtf \
		--highlight-style=haddock \
		--no-wrap \
		--self-contained --smart |sed 's/\\line\( [^{]\)/\1/g' >$@

$(PKGNAME)-$(BUILD_VERSION).pkg: pkg/distribution.xml \
                                 pkg/ch.roe.xnumon.pkg \
                                 pkg/ch.roe.kext.xnumon.pkg \
                                 pkg/resources/license.html \
                                 pkg/resources/readme.rtf $(MKFS)
	productbuild --distribution pkg/distribution.xml \
	             --resources pkg/resources \
	             --package-path pkg/ \
	             --identifier ch.roe.xnumon \
	             --version $(BUILD_VERSION) \
	             $(PRODUCTBUILDFLAGS) \
	             $@
	rm -f pkg/ch.roe.xnumon.pkg pkg/ch.roe.kext.xnumon.pkg
	rm -f pkg/resources/license.html pkg/resources/readme.rtf

pkg/ch.roe.xnumon.pkg: xnumon.signed \
                       mklaunchdplist \
                       pkg/newsyslog.conf \
                       pkg/uninstall.sh \
                       pkg/scripts/* \
                       $(MKFS)
	rm -rf pkgroot~
	mkdir -p pkgroot~/usr/local/sbin
	cp xnumon.signed pkgroot~/usr/local/sbin/xnumon
	mkdir -p pkgroot~/usr/local/bin
	cp pkg/xnumonctl.sh pkgroot~/usr/local/bin/xnumonctl
	mkdir -p pkgroot~/Library/LaunchDaemons
	./mklaunchdplist -l ch.roe.xnumon \
	          -d pkgroot~/Library/LaunchDaemons \
	          -e /usr/local/sbin/xnumon \
	          -- -d
	mkdir -p pkgroot~/private/etc/newsyslog.d
	cp pkg/newsyslog.conf \
	   pkgroot~/private/etc/newsyslog.d/ch.roe.xnumon.conf
	mkdir -p "pkgroot~/Library/Application Support/ch.roe.xnumon/"
	chmod 750 "pkgroot~/Library/Application Support/ch.roe.xnumon/"
	cat pkg/configuration.plist-default.in \
	   | sed 's/__BUILD_VERSION__/$(BUILD_VERSION)/' \
	   > "pkgroot~/Library/Application Support/ch.roe.xnumon/configuration.plist-default"
	chmod 440 "pkgroot~/Library/Application Support/ch.roe.xnumon/configuration.plist-default"
	cp pkg/uninstall.sh \
	   "pkgroot~/Library/Application Support/ch.roe.xnumon/uninstall.sh"
	chmod 550 "pkgroot~/Library/Application Support/ch.roe.xnumon/uninstall.sh"
	pkgbuild --root pkgroot~ \
	         --scripts pkg/scripts/ \
	         --identifier ch.roe.xnumon \
	         --version $(BUILD_VERSION) \
	         --ownership recommended \
	         $@
	rm -rf pkgroot~

pkg/ch.roe.kext.xnumon.pkg: kext \
                            pkg/scripts-kext/* \
                            $(MKFS)
	rm -rf pkgroot~
	mkdir -p pkgroot~/Library/Extensions
	cp -r kext/xnumon.kext pkgroot~/Library/Extensions/xnumon.kext
	pkgbuild --root pkgroot~ \
	         --scripts pkg/scripts-kext/ \
	         --identifier ch.roe.kext.xnumon \
	         --version $(BUILD_VERSION) \
	         --ownership recommended \
	         $@
	rm -rf pkgroot~

pkgclean:
	rm -f $(PKGNAME)-*.pkg

realclean: clean kextclean pkgclean

maintclean: realclean
	rm -rf $(EXTSRCS)

todo:
	@grep -r 'XXX\|TODO\|FIXME' .|grep -v __EXCLUDE__ || true

FORCE:

.PHONY: all sign copyright fetch clean test \
        kext kextclean \
        pkg pkgclean \
        realclean maintclean

