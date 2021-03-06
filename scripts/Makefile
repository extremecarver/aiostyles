#
#
# This is the makefile, that is used to build the All in one Garmin Map at the Openstreetmapserver and copy the result to the webdir for download.
# There are targets for every part (overlay) of the AIO-Map like basemap, fixme, addr...
# 
# 
#

SHELL := /bin/bash

AIOPATH := /osm/garmin/aio
PATH_TO_REGIONS := $(AIOPATH)/regions

# default region is europe
ifndef REGION
REGION := europe
endif

BUNDESLAENDER :=baden-wuerttemberg bayern berlin brandenburg bremen hamburg hessen mecklenburg-vorpommern niedersachsen nordrhein-westfalen rheinland-pfalz saarland sachsen-anhalt sachsen schleswig-holstein thueringen
COUNTRIES :=germany austria switzerland france italy united_kingdom albania andorra azores belarus belgium bosnia-herzegovina bulgaria croatia cyprus czech_republic denmark estonia finland greece hungary iceland isle_of_man kosovo latvia liechtenstein lithuania luxembourg macedonia malta moldova monaco montenegro netherlands norway poland portugal romania serbia slovakia slovenia spain sweden turkey ukraine
REGIONLIST := $(COUNTRIES) $(BUNDESLAENDER)
ALL_REGIONS :=canary_islands haiti europe $(REGIONLIST)
CS_ALL_REGIONS := $(shell echo $(ALL_REGIONS)|sed 's/ /,/g')


LAYER:=gbasemap,gaddr,gfixme,gmaxspeed,gosb,gboundary,gkeepright,gdamage,gcontourlines


# The Region should get extracted from Europe if it is part of it.
# The Regionlist defines which countries that are.
ifndef IS_PART_OF
ifeq ($(REGION),$(filter $(REGION),$(REGIONLIST)))
IS_PART_OF := europe
else
IS_PART_OF := false
endif
endif






# if we want to start some compressing tasks in a subshell to work parallel we use & instead of ; to separate shell commands
ifeq ($(WORK_PARALLEL),false)
USE_CORES :=1
SHELL_EXECUTOR := ;
else
USE_CORES :=4
SHELL_EXECUTOR := &
endif



# The Printfile defines where to store all the Output of the programs. I don't want to see everything when I get a mail from cron.
ifndef PRINTFILE
PRINTFILE := $(shell tempfile)
endif

ifeq ($(REGION),$(filter $(REGION),$(BUNDESLAENDER)))
DATAPATH := /osm/geofabrik-extrakte/europe/germany/$(REGION).osm.bz2
KURZ := DE
endif

ifeq ($(REGION),$(filter $(REGION),$(COUNTRIES)))
DATAPATH := /osm/geofabrik-extrakte/europe/$(REGION).osm.bz2
KURZ := EU
endif

ifeq ($(REGION),germany)
DATAPATH := /osm/geofabrik-extrakte/europe/germany.osm.bz2
KURZ := DE
endif

ifeq ($(REGION),united_kingdom)
DATAPATH := /osm/geofabrik-extrakte/europe/great_britain.osm.bz2
KURZ := UK
endif

ifeq ($(REGION),haiti)
DATAPATH := /osm/garmin/aio/regions/haiti/raw_data/haiti.osm.bz2
KURZ := HT
endif

ifeq ($(REGION),canary_islands)
DATAPATH := /osm/geofabrik-extrakte/africa/canary_islands.osm.bz2
KURZ := CI
endif

ifeq ($(REGION),europe)
DATAPATH := /osm/geofabrik-extrakte/europe.osm.bz2
KURZ := EU
endif

# Use the first 4 numbers of the mapid as Prefix and the 5th to distinguish between the layers. The last 3 numbers are to number the tiles.
# 6324 0 023 is the first Tile of the basemap.

ifndef TILE_PREFIX
TILE_PREFIX := 6324
endif
# it is hacker style to start with 23 ;)
ifndef TILE_START
TILE_START := 023
endif

#if we use existing tiles from europe we get the right one from the postgis database
ifneq ($(IS_PART_OF),false)
ifeq ($(REGION),$(filter $(REGION),$(BUNDESLAENDER)))
REGION_TILE_INDEX=$(shell psql -d aio -c "SELECT DISTINCT tiles$(1)$(IS_PART_OF).id FROM tiles$(1)$(IS_PART_OF),bundeslaender WHERE ST_Intersects(tiles$(1)$(IS_PART_OF).the_geom,bundeslaender.the_geom) AND LOWER(bundeslaender.name)=LOWER('$(REGION)');" | sed -n 's/[^0-9]*$(TILE_PREFIX)0\([0-9]*\).*/\1/gp' | tr '\n' ',' | sed 's/,$$//')
else
REGION_TILE_INDEX=$(shell psql -d aio -c "SELECT DISTINCT tiles$(1)$(IS_PART_OF).id FROM tiles$(1)$(IS_PART_OF),countries WHERE ST_Intersects(tiles$(1)$(IS_PART_OF).the_geom,countries.the_geom) AND LOWER(countries.country)=LOWER(REPLACE(REPLACE('$(REGION)','_',' '),'-',' and '));" | sed -n 's/[^0-9]*$(TILE_PREFIX)0\([0-9]*\).*/\1/gp' | tr '\n' ',' | sed 's/,$$//')
endif
REGION_TILE_INDEX_PIPES=$(shell echo "$(call REGION_TILE_INDEX,$(1))" | tr ',' '|')
endif


#BBOX := $(shell bzcat $(DATAPATH) |sed -n '4q;s/<bound box="/bottom=/;s/,/ left=/;s/,/ top=/;s/,/ right=/;s/" origin=.*//p')
# bottom left top right
# These choords are used for example to get all openstreetbugs in the bounding box of our region. We try to extract the bounds out of the geofabrik osm extract.
CHOORDS := $(shell bzcat $(DATAPATH) |sed -n '/<bound/{s/.*box="\([^"]*\)".*/\1/;s/,/ /gp;q}')
BOTTOM := $(word 1, $(CHOORDS) )
LEFT := $(word 2, $(CHOORDS) )
TOP := $(word 3, $(CHOORDS) )
RIGHT := $(word 4, $(CHOORDS) )
BBOX := bottom=$(BOTTOM) left=$(LEFT) top=$(TOP) right=$(RIGHT)

#OSMOSIS := $(AIOPATH)/osmosis-0.31/bin/osmosis
OSMOSIS := osmosis
REGIONPATH := $(PATH_TO_REGIONS)/$(REGION)
DATE := $(shell date +%Y%m%d)
WEBDIR := /osm/wwwroot/aio
GMAPTOOL := /usr/local/bin/gmt
JAVA_OPT := -Xmx8000M -ea -XX:MaxPermSize=256M
STYLEPATH := $(AIOPATH)/styles
TILEPATH := $(REGIONPATH)/tiles
MKGMAP := java $(JAVA_OPT) -jar $(AIOPATH)/mkgmap.jar
SPLITTER := java $(JAVA_OPT) -jar $(AIOPATH)/splitter.jar
LOGPATH := ${AIOPATH}/logfiles/${REGION}/$(DATE)

# Define dependencies of the layers.
ifeq ($(IS_PART_OF),false)
DEPENDENCY_BASEMAP := $(TILEPATH)/template.args
DEPENDENCY_ADDR := $(TILEPATH)/template.args
DEPENDENCY_FIXME := $(TILEPATH)/template.args
DEPENDENCY_MAXSPEED := $(TILEPATH)/template.args
DEPENDENCY_BOUNDARY := $(REGIONPATH)/raw_data/boundssplit/template.args
DEPENDENCY_KEEPRIGHT := $(REGIONPATH)/raw_data/keeprightsplit/template.args
DEPENDENCY_DAMAGE := $(TILEPATH)/template.args
else
PART_META_PATH := $(PATH_TO_REGIONS)/$(IS_PART_OF)
DEPENDENCY_BASEMAP := $(PART_META_PATH)/gmapsupps/gbasemap/gmapsupp.img
DEPENDENCY_ADDR := $(PART_META_PATH)/gmapsupps/gaddr/gmapsupp.img
DEPENDENCY_FIXME := $(PART_META_PATH)/gmapsupps/gfixme/gmapsupp.img
DEPENDENCY_MAXSPEED := $(PART_META_PATH)/gmapsupps/gmaxspeed/gmapsupp.img
DEPENDENCY_BOUNDARY := $(PART_META_PATH)/gmapsupps/gboundary/gmapsupp.img
DEPENDENCY_KEEPRIGHT := $(PART_META_PATH)/gmapsupps/gkeepright/gmapsupp.img
DEPENDENCY_DAMAGE := $(PART_META_PATH)/gmapsupps/gdamage/gmapsupp.img
endif

# We want other options for the overlays than for the basemaplayer containing the routing etc.
OPTIONS := --max-jobs=$(USE_CORES) --country-name=$(REGION) --country-abbr=$(KURZ) --area-name=$(KURZ) --latin1 --tdbfile --gmapsupp --nsis --keep-going
GBASEMAPOPTIONS := $(OPTIONS) --add-pois-to-areas --make-all-cycleways --link-pois-to-ways --remove-short-arcs --net --route --index  --generate-sea=polygons,no-sea-sectors,close-gaps=2000
NOBASEMAPOPTIONS := $(OPTIONS) --no-poi-address --no-sorted-roads --ignore-maxspeeds --ignore-turn-restrictions --ignore-osm-bounds --transparent

# The splitter can use an older areas.list file from previous computations by giving USE_OLD_AREAS_LIST=true. Default is to split new.
ifndef USE_OLD_AREAS_LIST
USE_OLD_AREAS_LIST:=false
else
ifeq ($(USE_OLD_AREAS_LIST),true)
ifndef AREAS_LIST_FILE
AREAS_LIST_FILE:=$(TILEPATH)/areas.list
endif
# We have to ensure that the areas.list file exist. if not we split completely new.
override USE_OLD_AREAS_LIST:=$(shell if test -f $(AREAS_LIST_FILE); then echo true; else echo false; fi)
endif
endif

# The Tilesplitter options:
GEONAMES:=--geonames-file=$(AIOPATH)/geonames/cities15000.zip
ifeq ($(USE_OLD_AREAS_LIST),true)
SPLITTER_OPTIONS := --split-file=$(AREAS_LIST_FILE) --cache=$(REGIONPATH)/raw_data/splittercache $(GEONAMES)
SPLITTER_OPTIONS_BOUNDARY := --split-file=$(REGIONPATH)/raw_data/boundssplit/areas.list $(GEONAMES)
SPLITTER_OPTIONS_KEEPRIGHT := --split-file=$(REGIONPATH)/raw_data/keeprightsplit/areas.list $(GEONAMES)
else
SPLITTER_OPTIONS := --mapid=$(TILE_PREFIX)0$(TILE_START) --max-nodes=1000000 --cache=$(REGIONPATH)/raw_data/splittercache $(GEONAMES)
SPLITTER_OPTIONS_BOUNDARY := --mapid=$(TILE_PREFIX)4$(TILE_START) --max-nodes=1000000 $(GEONAMES)
SPLITTER_OPTIONS_KEEPRIGHT := --mapid=$(TILE_PREFIX)7$(TILE_START) --max-nodes=200000 $(GEONAMES)
endif

# You can specify different Compressors for the result
# Default is gzip

COMPRESSOR:=gzip

ifeq ($(COMPRESSOR),zip)
COMPRESS_SUFFIX:=zip
endif
ifeq ($(COMPRESSOR),gzip)
COMPRESS_SUFFIX:=gz
endif
ifeq ($(COMPRESSOR),bzip2)
COMPRESS_SUFFIX:=bz2
endif
ifeq ($(COMPRESSOR),p7zip)
COMPRESS_SUFFIX:=7z
endif

tar_img_dir = \
	(tar -I $(COMPRESSOR) -cf $(REGIONPATH)/release/g$(1).${DATE}.tar.$(COMPRESS_SUFFIX) -C $(REGIONPATH) g$(1) && \
	cp -f $(REGIONPATH)/release/g$(1).$(DATE).tar.$(COMPRESS_SUFFIX) $(WEBDIR)/$(REGION)/$(DATE)/ && \
	ln -sf $(DATE)/g$(1).$(DATE).tar.$(COMPRESS_SUFFIX) $(WEBDIR)/$(REGION)/g$(1).tar.$(COMPRESS_SUFFIX) && \
	md5sum -b $(WEBDIR)/$(REGION)/g$(1).tar.$(COMPRESS_SUFFIX) > $(WEBDIR)/$(REGION)/g$(1).tar.$(COMPRESS_SUFFIX).md5 ) $(SHELL_EXECUTOR)

compress_gmapsupp = \
	($(COMPRESSOR) -c $(REGIONPATH)/gmapsupps/g$(1)/gmapsupp.img > $(WEBDIR)/$(REGION)/$(DATE)/g$(1).$(DATE).img.$(COMPRESS_SUFFIX) && \
	ln -sf $(DATE)/g$(1).$(DATE).img.$(COMPRESS_SUFFIX) $(WEBDIR)/$(REGION)/g$(1).img.$(COMPRESS_SUFFIX) && \
	md5sum -b $(WEBDIR)/$(REGION)/g$(1).img.$(COMPRESS_SUFFIX) > $(WEBDIR)/$(REGION)/g$(1).img.$(COMPRESS_SUFFIX).md5 ) $(SHELL_EXECUTOR)


ifeq ($(REGION),europe)
build_exe = \
	sed -i 's/^  File "\(.*\)/  CopyFiles "$$EXEDIR\\\1 $$INSTDIR/g' $(REGIONPATH)/g$(1)/osmmap.nsi && \
	makensis -O${LOGPATH}/makensis_$(1).log $(REGIONPATH)/g$(1)/osmmap.nsi ;
else
build_exe = \
	sed 's/^  File "\(.*\)/  CopyFiles "$$EXEDIR\\\1 $$INSTDIR/g' $(REGIONPATH)/g$(1)/osmmap.nsi > $(REGIONPATH)/g$(1)/osmmap_copy.nsi && \
	makensis -O${LOGPATH}/makensis_$(1)_copy.log $(REGIONPATH)/g$(1)/osmmap_copy.nsi ; \
	(sed -i 's|^OutFile "|OutFile "$(REGIONPATH)/gmapsupps/g$(1)/|' $(REGIONPATH)/g$(1)/osmmap.nsi && \
	makensis -O${LOGPATH}/makensis_$(1).log $(REGIONPATH)/g$(1)/osmmap.nsi && \
	mv $(REGIONPATH)/gmapsupps/g$(1)/*.exe $(WEBDIR)/$(REGION)/$(DATE)/OSM-AllInOne-$(KURZ)-$(1).$(DATE).exe && \
	ln -sf $(DATE)/OSM-AllInOne-$(KURZ)-$(1).$(DATE).exe $(WEBDIR)/$(REGION)/OSM-AllInOne-$(KURZ)-$(1).exe && \
	md5sum -b $(WEBDIR)/$(REGION)/OSM-AllInOne-$(KURZ)-$(1).exe > $(WEBDIR)/$(REGION)/OSM-AllInOne-$(KURZ)-$(1).exe.md5 ) $(SHELL_EXECUTOR)
endif


# params:
# name,options,source
do_stuff = \
	mkdir -p $(LOGPATH) ; \
	cd $(REGIONPATH)/g$(1)/ && \
	cp $(STYLEPATH)/$(1).TYP $(REGIONPATH)/g$(1) && \
	/usr/bin/time -o $(LOGPATH)/time_mkgmap_$(1) $(MKGMAP) --style-file=$(STYLEPATH)/$(1)_style --series-name="OSM-AllInOne-$(KURZ)-$(1)" $(2) $(3) $(1).TYP 2> $(LOGPATH)/mkgmap_$(1).log && \
	{ mkdir -p $(WEBDIR)/$(REGION)/$(DATE) && \
	$(call build_exe,$(1)) \
	mv $(REGIONPATH)/g$(1)/gmapsupp.img $(REGIONPATH)/gmapsupps/g$(1)/gmapsupp.img && \
	(  $(call tar_img_dir,$(1)) \
	$(call compress_gmapsupp,$(1)) ) ; \
	echo -e "Parameters used with mkgmap to build the $(1)-Layer:\n--style-file=$(1)_style --series-name=\"OSM-AllInOne-$(KURZ)-$(1)\" $(2) $(3)\nmkgmap --version:" > $(WEBDIR)/$(REGION)/mkgmap_params_$(1) ; $(MKGMAP) --version 2>> $(WEBDIR)/$(REGION)/mkgmap_params_$(1) ; } ;


# if we reuse made tiles from bigger areas we have to copy them into our directory and make a new template.args containing only the needed (and copied) tiles
copy_tiles = \
	cp -a $(PART_META_PATH)/$(1)/$(TILE_PREFIX)*{$(call REGION_TILE_INDEX,$(2))}.img $(REGIONPATH)/$(1)/ ; \
	grep -A2 -E "mapname: $(TILE_PREFIX)[0-9]($(call REGION_TILE_INDEX_PIPES,$(2)))" $(PART_META_PATH)/tiles/$(1)_template.args | sed 's/^--$$//g;/mapname:/h;/input-file:/{g;s/mapname: \([0-9]*\)/input-file: \1.img/}' > $(REGIONPATH)/$(1)/template.args

# bring a given areas.list to a specified postgis database table

areas_to_db = \
	psql -d aio -c "DELETE FROM $(2);" ; \
	sed -n '/^[0-9]*:/{N;s/^\([0-9]*\):[^:]*: \(.*\),\(.*\) to \(.*\),\(.*\)/INSERT INTO $(2) (id,the_geom) VALUES (\1,ST_SetSRID(ST_MakeBox2D(ST_Point(\3,\2),ST_Point(\5,\4)),4326));/p}' $(1) | psql -d aio


ifeq ($(REGION),europe)
ifeq ($(USE_OLD_AREAS_LIST),false)
define prepare_areas
	$(call areas_to_db,$(1)/areas.list,tiles$(2)europe)
	$(AIOPATH)/makekml.sh tiles$(2)europe
endef
endif
endif

define process_splitter_files
# Set the full path name for the tiles in template.args
	sed -i "s|input-file: \(.*\)|input-file: $(1)/\1|g" $(1)/template.args
	$(call prepare_areas,$(1),$(2))
endef

# make_layer takes the following parameters $(1)-$(9) that differ for every layer:
# params: 	name		mkgmap_options		layernumber used for TileIDs	family-id	product-id	draw-priority	template.args						PG_TABLE_SELECTOR	extra_opts while combine premade tiles
# example1: 	basemap		$(GBASEMAPOPTIONS)	0				4		45		20		$(TILEPATH)/template.args				_			--index --route
# example2:	boundary	$(NOBASEMAPOPTIONS)	4				6		30		21		$(REGIONPATH)/raw_data/boundssplit/template.args	_boundary_
ifeq ($(IS_PART_OF),false)
handle_layer = \
	sed 's/mapname: $(TILE_PREFIX)[0-9]/mapname: $(TILE_PREFIX)$(3)/g;s/description: \(.*\)/description: \1-$(1)/g' $(7) > $(TILEPATH)/g$(1)_template.args && \
	$(call do_stuff,$(1),$(2) --family-id=$(4) --product-id=$(5) --family-name=$(1) --draw-priority=$(6),-c $(TILEPATH)/g$(1)_template.args)
else
handle_layer = \
	$(call copy_tiles,g$(1),$(8)) && \
	$(call do_stuff,$(1),$(9) --gmapsupp --nsis --family-id=$(4) --product-id=$(5) --family-name=$(1),-c $(REGIONPATH)/g$(1)/template.args)
endif


make_layer = \
	rm -f $(REGIONPATH)/g$(1)/* ; \
	echo -e "To install this stuff in Mapsource you have do execute the .exe from this directory.\nThis will start the installer, copy the *.img's to your Mapsource-map-folder and write to your registry.\n" >> $(REGIONPATH)/g$(1)/HOWTO.txt ; \
	echo -e "To load this map in QLandkarteGT you have to select the osmmap.tdb in this directory from within QLandKarteGt.\nThe Typfile in this directory gets automatically included.\n" >> $(REGIONPATH)/g$(1)/HOWTO.txt ; \
	echo -e "For generating a gmapsupp to copy to your SD-Card you need to run mkgmap like this:\njava -ea -jar mkgmap.jar --gmapsupp --family-id=$(4) --product-id=$(5) --family-name=$(1) ./*.img $(1).TYP\n" >> $(REGIONPATH)/g$(1)/HOWTO.txt ; \
	echo -e "Enjoy!\n" >> $(REGIONPATH)/g$(1)/HOWTO.txt ; \
	$(call handle_layer,$(1),$(2),$(3),$(4),$(5),$(6),$(7),$(8),$(9))

all: $(WEBDIR)/$(REGION)/gmapsupp.img.bz2 $(WEBDIR)/$(REGION)/styles.tar.bz2


define make_dirs
	mkdir -p $(1)/{$(LAYER)}
	mkdir -p $(1)/raw_data/{splittercache,boundssplit,keeprightsplit,contourlines}
	mkdir -p $(1)/gmapsupps/{$(LAYER)}
	mkdir -p $(1)/tiles
	mkdir -p $(1)/release
endef

# create the structure for a given region

structure:
	$(call make_dirs,$(REGIONPATH))

# create the structure for all known regions

restructure:
	$(call make_dirs,$(PATH_TO_REGIONS)/{$(CS_ALL_REGIONS)})


# Clean the webdir and remove all directories that are older than EXPIRE date.
# Clean the release dir of the region-dirs.

ifndef EXPIRE
EXPIRE := 6
endif

clean:
	rm -f $(PATH_TO_REGIONS)/{$(CS_ALL_REGIONS)}/release/*.gz ; \
	find $(WEBDIR)/{$(CS_ALL_REGIONS)} -type d -ctime +$(EXPIRE) -exec rm -rf '{}' ';'

# Clean all stuff not directly needed in region dirs
# Warning you need the imgs of europe to create the other subcountries! 
extreme_clean: clean
	rm -f $(PATH_TO_REGIONS)/{$(CS_ALL_REGIONS)}/{$(LAYER)}/*.img ; \
	rm -f $(PATH_TO_REGIONS)/{$(CS_ALL_REGIONS)}/tiles/*.osm.gz

	

# If something has changed in the stylepath repack it and write it to the webdir.

$(WEBDIR)/$(REGION)/styles.tar.bz2 : $(STYLEPATH)/*/*
	tar cjf $(WEBDIR)/$(REGION)/styles.tar.bz2 -C $(STYLEPATH)/../ styles


# Generate the gmapsupp by combining all layer-gmapsupp.img-files.

$(WEBDIR)/$(REGION)/gmapsupp.img.bz2 : $(REGIONPATH)/gmapsupps/gbasemap/gmapsupp.img $(REGIONPATH)/gmapsupps/gkeepright/gmapsupp.img $(REGIONPATH)/gmapsupps/gaddr/gmapsupp.img $(REGIONPATH)/gmapsupps/gfixme/gmapsupp.img $(REGIONPATH)/gmapsupps/gmaxspeed/gmapsupp.img $(REGIONPATH)/gmapsupps/gboundary/gmapsupp.img | $(REGIONPATH)/gmapsupps/gosb/gmapsupp.img
# This is an OR Statemant in Makefilesyntax:
ifeq ($(REGION),$(filter $(REGION),$(BUNDESLAENDER) $(COUNTRIES) europe))
# include contourlines of germany if REGION is germany or europe
ifeq ($(REGION),$(filter $(REGION),germany europe))
	cd $(REGIONPATH)/release; $(MKGMAP) --gmapsupp $(REGIONPATH)/gmapsupps/g{basemap,addr,fixme,osb,boundary,maxspeed}/gmapsupp.img $(PATH_TO_REGIONS)/germany/gmapsupps/ghoehe/gmapsupp.img
else
	cd $(REGIONPATH)/release; $(MKGMAP) --gmapsupp $(REGIONPATH)/gmapsupps/g{basemap,addr,fixme,osb,boundary,maxspeed,keepright}/gmapsupp.img
endif
else
ifeq ($(REGION),$(filter $(REGION),haiti))
#	$(GMAPTOOL) -j -m $(DATE) -o $(REGIONPATH)/release/gmapsupp.img $(REGIONPATH)/gmapsupps/gbasemap/gmapsupp.img $(REGIONPATH)/gmapsupps/gaddr/gmapsupp.img $(REGIONPATH)/gmapsupps/gosb/gmapsupp.img $(REGIONPATH)/gmapsupps/gboundary/gmapsupp.img $(REGIONPATH)/gmapsupps/gdamage/gmapsupp.img
	cd $(REGIONPATH)/release; $(MKGMAP) --gmapsupp $(REGIONPATH)/gmapsupps/g{basemap,addr,fixme,osb,boundary,damage,keepright}/gmapsupp.img
	cp $(REGIONPATH)/release/gmapsupp.img $(WEBDIR)/$(REGION)/gmapsupp.img
else
	cd $(REGIONPATH)/release; $(MKGMAP) --gmapsupp $(REGIONPATH)/gmapsupps/g{basemap,addr,fixme,osb,boundary,maxspeed}/gmapsupp.img

#	$(GMAPTOOL) -j -m $(DATE) -o $(REGIONPATH)/release/gmapsupp.img $(REGIONPATH)/gmapsupps/gbasemap/gmapsupp.img $(REGIONPATH)/gmapsupps/gaddr/gmapsupp.img $(REGIONPATH)/gmapsupps/gfixme/gmapsupp.img $(REGIONPATH)/gmapsupps/gosb/gmapsupp.img $(REGIONPATH)/gmapsupps/gboundary/gmapsupp.img $(REGIONPATH)/../germany/gmapsupps/ghoehe/gmapsupp.img
endif
endif
	$(COMPRESSOR) -f $(REGIONPATH)/release/gmapsupp.img
	mv $(REGIONPATH)/release/gmapsupp.img.$(COMPRESS_SUFFIX) $(REGIONPATH)/release/gmapsupp.img.$(DATE).$(COMPRESS_SUFFIX)
	cp -f $(REGIONPATH)/release/gmapsupp.img.$(DATE).$(COMPRESS_SUFFIX) $(WEBDIR)/$(REGION)/$(DATE)/
	ln -sf $(WEBDIR)/$(REGION)/$(DATE)/gmapsupp.img.$(DATE).$(COMPRESS_SUFFIX) $(WEBDIR)/$(REGION)/gmapsupp.img.$(COMPRESS_SUFFIX)
	md5sum -b $(WEBDIR)/$(REGION)/gmapsupp.img.$(COMPRESS_SUFFIX) > $(WEBDIR)/$(REGION)/gmapsupp.img.$(COMPRESS_SUFFIX).md5 &
ifeq ($(REGION),germany)
	ncftpput -b -f ~/.ncftp/juergen gmapsupps/germany $(WEBDIR)/$(REGION)/*.$(COMPRESS_SUFFIX)
endif
	echo '--------------------------------------- Ende $(REGION)' >> $(PRINTFILE);date >> $(PRINTFILE);echo "-----------------------------------" >> $(PRINTFILE)

$(REGIONPATH)/gmapsupps/gbasemap/gmapsupp.img : $(DEPENDENCY_BASEMAP)
	$(call make_layer,basemap,$(GBASEMAPOPTIONS),0,4,45,10,$(TILEPATH)/template.args,_,--index --route)

$(REGIONPATH)/gmapsupps/gaddr/gmapsupp.img : $(DEPENDENCY_ADDR)
	$(call make_layer,addr,$(NOBASEMAPOPTIONS) --add-pois-to-areas,1,5,40,20,$(TILEPATH)/template.args,_)

$(REGIONPATH)/gmapsupps/gfixme/gmapsupp.img : $(DEPENDENCY_FIXME)
	$(call make_layer,fixme,$(NOBASEMAPOPTIONS),2,3,33,22,$(TILEPATH)/template.args,_)

$(REGIONPATH)/gmapsupps/gmaxspeed/gmapsupp.img : $(DEPENDENCY_MAXSPEED)
	$(call make_layer,maxspeed,$(NOBASEMAPOPTIONS),6,84,15,19,$(TILEPATH)/template.args,_)

$(REGIONPATH)/gmapsupps/gboundary/gmapsupp.img : $(DEPENDENCY_BOUNDARY)
	$(call make_layer,boundary,$(NOBASEMAPOPTIONS),4,6,30,21,$(REGIONPATH)/raw_data/boundssplit/template.args,_boundary_)

$(REGIONPATH)/gmapsupps/gosb/gmapsupp.img : $(AIOPATH)/openstreetbugs/osbdump_latest.sql.bz2
	bzcat $(AIOPATH)/openstreetbugs/osbdump_latest.sql.bz2 | $(AIOPATH)/osbsql2osm | $(OSMOSIS) --rx - --bb $(BBOX) --nkv keyValueList="type.0" --wx $(REGIONPATH)/raw_data/osb_$(REGION).osm
	rm -f $(REGIONPATH)/gosb/*
	$(call do_stuff,osb,$(NOBASEMAPOPTIONS) --family-id=2323 --product-id=42 --family-name=osb  --mapname=$(TILE_PREFIX)3$(TILE_START) --description='Openstreetbugs' --draw-priority=23,../raw_data/osb_$(REGION).osm)

$(REGIONPATH)/gmapsupps/gkeepright/gmapsupp.img : $(DEPENDENCY_KEEPRIGHT)
	$(call make_layer,keepright,$(NOBASEMAPOPTIONS),7,777,777,24,$(REGIONPATH)/raw_data/keeprightsplit/template.args,_keepright_)


$(REGIONPATH)/gmapsupps/gdamage/gmapsupp.img : $(DEPENDENCY_DAMAGE)
	$(call make_layer,damage,$(NOBASEMAPOPTIONS),5,4242,2323,25,$(TILEPATH)/template.args,_)


# contourline target not in use:
$(REGIONPATH)/gmapsupps/gcontourlines/gmapsupp.img:
	cd $(REGIONPATH)/raw_data/contourlines; phyghtmap -a $(LEFT):$(BOTTOM):$(RIGHT):$(TOP) -j $(USE_CORES)
	cd $(REGIONPATH)/gcontourlines; java $(JAVA_OPT) -jar $(AIOPATH)/mkgmap.jar $(NOBASEMAPOPTIONS) --style-file=$(STYLEPATH)/basemap_style --description="contour lines" --family-id=5 --product-id=25  --series-name="OSM-AllInOne-$(KURZ)-contourlines" --family-name=contourlines --mapname=63248345 --draw-priority=100  $(REGIONPATH)/raw_data/contourlines/*.osm ;\
	makensis osmmap.nsi
	cp $(REGIONPATH)/gcontourlines/OSM-AllInOne-$(KURZ)-contourlines.exe $(WEBDIR)/$(REGION)/
	mv $(REGIONPATH)/gcontourlines/gmapsupp.img $(REGIONPATH)/gmapsupps/gcontourlines/gmapsupp.img
	bzip2 -c $(REGIONPATH)/gmapsupps/gcontourlines/gmapsupp.img > $(WEBDIR)/$(REGION)/contourlines.img.bz2
	tar cjf $(REGIONPATH)/release/gcontourlines.tar.bz2 -C $(REGIONPATH) gcontourlines
	cp -f $(REGIONPATH)/release/gcontourlines.tar.bz2 $(WEBDIR)/$(REGION)/
	md5sum -b $(WEBDIR)/$(REGION)/gcontourlines.tar.bz2 > $(WEBDIR)/$(REGION)/gcontourlines.tar.bz2.md5

$(REGIONPATH)/raw_data/keepright.osm.gz : $(AIOPATH)/keepright/keepright_errors.txt.bz2
	bzcat $(AIOPATH)/keepright/keepright_errors.txt.bz2 | awk -f $(AIOPATH)/keepright2osm.awk | $(OSMOSIS) --rx - --bb $(BBOX) --wx $(REGIONPATH)/raw_data/keepright.osm.gz

$(REGIONPATH)/raw_data/keeprightsplit/template.args : $(REGIONPATH)/raw_data/keepright.osm.gz
	cd $(REGIONPATH)/raw_data/keeprightsplit/ && rm *.osm.gz ; $(SPLITTER) $(SPLITTER_OPTIONS_KEEPRIGHT) $(REGIONPATH)/raw_data/keepright.osm.gz 2> $(LOGPATH)/splitter_keepright.log
	$(call process_splitter_files,$(REGIONPATH)/raw_data/keeprightsplit,_keepright_)

$(REGIONPATH)/raw_data/boundssplit/template.args : $(TILEPATH)/template.args
	cd $(REGIONPATH)/raw_data/boundssplit/ && rm *.osm.gz ; $(SPLITTER) $(SPLITTER_OPTIONS_BOUNDARY) $(REGIONPATH)/raw_data/bounds.osm.gz 2> $(LOGPATH)/splitter_boundary.log
	$(call process_splitter_files,$(REGIONPATH)/raw_data/boundssplit,_boundary_)

$(TILEPATH)/template.args : $(DATAPATH)
	echo "$(REGION) Auspacken Start:" >> $(PRINTFILE);date >> $(PRINTFILE)
	mkdir -p $(LOGPATH)
	cd $(TILEPATH)/ && rm *.osm.gz ; /usr/bin/time -o $(LOGPATH)/time_splitter bzcat $(DATAPATH) | tee \
	>($(OSMOSIS) --rx - --way-key-value keyValueList="boundary.administrative,boundary.national,boundary.political" --used-node --write-xml $(REGIONPATH)/raw_data/bounds.osm.gz) \
	| $(SPLITTER) $(SPLITTER_OPTIONS) /dev/stdin 2> $(LOGPATH)/splitter.log
	echo "USE_OLD_AREAS_LIST=$(USE_OLD_AREAS_LIST)" >> $(LOGPATH)/time_splitter
	$(call process_splitter_files,$(TILEPATH),_)
