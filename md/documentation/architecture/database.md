# Database (outdated)


## ER model

Below are ER models of views and example layers and roles.

![maplayer](/images/architecture/maplayer-ER.png)

![views](/images/architecture/tablesER.png)

![rolesetc](/images/architecture/tables2ER.png)




## Table create scripts

namefi, namesv and nameen columns are deprecated, use locale column, which will contain localization in i18n.

This table contains maplayers. Layerclassid indicates the layer's class position in the layer hierarchy.

### Maplayer


  	-- DROP TABLE portti_maplayer IF EXISTS;

  	CREATE TABLE portti_maplayer
  	(
  	id serial NOT NULL,
  	layerclassid integer,
  	namefi character varying(2000),
  	namesv character varying(2000),
  	nameen character varying(2000),
  	wmsname character varying(2000),
  	wmsurl character varying(2000),
    opacity integer,
    style text,
    minscale double precision,
    maxscale double precision,
    description_link character varying(2000),
    legend_image character varying(2000),
    inspire_theme_id integer,
    dataurl character varying(2000),
    metadataurl character varying(2000),
    order_number integer,
    layer_type character varying(100) NOT NULL,
    tile_matrix_set_id character varying(1024),
    tile_matrix_set_data text,
    created timestamp with time zone,
    updated timestamp with time zone,
    wms_dcp_http character varying(2000),
    wms_parameter_layers character varying(2000),
    resource_url_scheme character varying(100),
    resource_url_scheme_pattern character varying(2000),
    resource_url_client_pattern character varying(2000),
    resource_daily_max_per_ip integer,
    xslt text,
    gfi_type character varying(2000),
    subtitle_fi character varying(2000),
    subtitle_en character varying(2000),
    subtitle_sv character varying(2000),
    selection_style text,
    version character varying(10),
    epsg integer DEFAULT 3067,
    locale text,
    CONSTRAINT portti_maplayer_pkey PRIMARY KEY (id),
    CONSTRAINT portti_maplayer_inspire_theme_id_fkey FOREIGN KEY (inspire_theme_id)
        REFERENCES portti_inspiretheme (id) MATCH SIMPLE
        ON UPDATE NO ACTION ON DELETE SET NULL,
    CONSTRAINT portti_maplayer_layerclassid_fkey FOREIGN KEY (layerclassid)
        REFERENCES portti_layerclass (id) MATCH SIMPLE
        ON UPDATE NO ACTION ON DELETE CASCADE
        )
    WITH (
    OIDS=FALSE
    );
    ALTER TABLE portti_maplayer
    GRANT SELECT, UPDATE ON TABLE portti_maplayer TO maplog;

    -- Index: portti_maplayer_q1

    -- DROP INDEX portti_maplayer_q1;

    CREATE INDEX portti_maplayer_q1
    ON portti_maplayer
    USING btree
    (layerclassid);

    -- Index: portti_maplayer_q2

    -- DROP INDEX portti_maplayer_q2;

    CREATE INDEX portti_maplayer_q2
    ON portti_maplayer
    USING btree
    (inspire_theme_id);

    -- Index: portti_maplayer_q3

    -- DROP INDEX portti_maplayer_q3;

    CREATE INDEX portti_maplayer_q3
    	ON portti_maplayer
    	USING btree
  	  (order_number);


### Layerclass

LayerClass specifies the layer's organization for normal layers, basemaps and groupmaps. If the layer is a basemap or a groupmap it has parent which refers to another layerclass. 





    -- DROP TABLE portti_layerclass;

    CREATE TABLE portti_layerclass
    (
    id serial NOT NULL,
    namefi character varying(2000),
    namesv character varying(2000),
    nameen character varying(2000),
    maplayers_selectable boolean,
    parent integer,
    legend_image character varying(2000),
    dataurl character varying(2000),
    group_map boolean,
    locale text,
    CONSTRAINT portti_layerclass_pkey PRIMARY KEY (id),
    CONSTRAINT portti_layerclass_parent_fkey FOREIGN KEY (parent)
        REFERENCES portti_layerclass (id) MATCH SIMPLE
        ON UPDATE NO ACTION ON DELETE CASCADE
    )
    WITH (
    OIDS=FALSE
    );


    -- Index: portti_layerclass_q1

    -- DROP INDEX portti_layerclass_q1;

    CREATE INDEX portti_layerclass_q1
    ON portti_layerclass
    USING btree
    (parent);

### InspireTheme

This table tells the layers inspire name.


	-- Table: portti_inspiretheme

	-- DROP TABLE portti_inspiretheme;

	CREATE TABLE portti_inspiretheme
	(
	  id serial NOT NULL,
	  namefi character varying(2000),
	  namesv character varying(2000),
	  nameen character varying(2000),
	  CONSTRAINT portti_inspiretheme_pkey PRIMARY KEY (id)
	)
	WITH (
	  OIDS=FALSE
	);



### Maplayer metadata

This table specifies layers bbox.

    -- Table: portti_maplayer_metadata

	-- DROP TABLE portti_maplayer_metadata;

	CREATE TABLE portti_maplayer_metadata
	(
	  id serial NOT NULL,
	  maplayerid integer,
	  uuid character varying(256),
	  namefi character varying(512),
	  namesv character varying(512),
	  nameen character varying(512),
	  abstractfi text,
	  abstractsv text,
	  abstracten text,
	  browsegraphic character varying(1024),
	  geom character varying(512),
	  CONSTRAINT portti_maplayer_metadata_pkey PRIMARY KEY (id)
	)
	WITH (
	  OIDS=FALSE
	);
	ALTER TABLE portti_maplayer_metadata
	  OWNER TO postgres;
	GRANT ALL ON TABLE portti_maplayer_metadata TO postgres;
	GRANT SELECT, UPDATE, INSERT, DELETE, REFERENCES ON TABLE portti_maplayer_metadata TO maplog;


	-- Index: portti_maplayer_metadata_q1

	-- DROP INDEX portti_maplayer_metadata_q1;

	CREATE INDEX portti_maplayer_metadata_q1
	  ON portti_maplayer_metadata
	  USING btree
	  (maplayerid);

	-- Index: portti_maplayer_metadata_q2

	-- DROP INDEX portti_maplayer_metadata_q2;

	CREATE INDEX portti_maplayer_metadata_q2
	  ON portti_maplayer_metadata
	  USING btree
	  (uuid);


### Backend status

This table indicates whether the layer's server is online. This is updated in Paikkatietoikkuna as a cron process.

	-- Table: portti_backendstatus

	-- DROP TABLE portti_backendstatus;

	CREATE TABLE portti_backendstatus
	(
	  id bigserial NOT NULL,
	  ts timestamp without time zone DEFAULT now(),
	  maplayer_id character varying(50),
	  status character varying(500),
	  statusmessage character varying(2000),
	  infourl character varying(2000),
	  statusjson text,
	  CONSTRAINT portti_backendstatus_pkey PRIMARY KEY (id)
	)
	WITH (
	  OIDS=FALSE
	);
	ALTER TABLE portti_backendstatus
	  OWNER TO postgres;
	GRANT ALL ON TABLE portti_backendstatus TO postgres;
	GRANT SELECT, UPDATE, INSERT, TRUNCATE, DELETE ON TABLE portti_backendstatus TO maplog;


	-- Index: portti_backendstatus_maplayer_id_q1

	-- DROP INDEX portti_backendstatus_maplayer_id_q1;

	CREATE INDEX portti_backendstatus_maplayer_id_q1
	  ON portti_backendstatus
	  USING btree
	  (maplayer_id);

	-- Index: portti_backendstatus_maplayer_id_q2

	-- DROP INDEX portti_backendstatus_maplayer_id_q2;

	CREATE INDEX portti_backendstatus_maplayer_id_q2
	  ON portti_backendstatus
	  USING btree
	  (maplayer_id, status);

	-- Index: portti_backendstatus_maplayer_id_q3

	-- DROP INDEX portti_backendstatus_maplayer_id_q3;

	CREATE INDEX portti_backendstatus_maplayer_id_q3
	  ON portti_backendstatus
	  USING btree
	  (maplayer_id, status, ts);

	-- Index: portti_backendstatus_maplayer_id_q4

	-- DROP INDEX portti_backendstatus_maplayer_id_q4;

	CREATE INDEX portti_backendstatus_maplayer_id_q4
	  ON portti_backendstatus
	  USING btree
	  (status);

### Capabilities cache

Cababilities requests are stored here and updated by a cron process.

	-- Table: portti_capabilities_cache

	-- DROP TABLE portti_capabilities_cache;

	CREATE TABLE portti_capabilities_cache
	(
	  layer_id integer NOT NULL,
	  data text,
	  updated timestamp without time zone,
	  "WMSversion" character(10) NOT NULL,
	  CONSTRAINT portti_capabilities_cache_pkey PRIMARY KEY (layer_id)
	)
	WITH (
	  OIDS=FALSE
	);




	-- Trigger: update_capabilities_cache_timestamp on portti_capabilities_cache

	-- DROP TRIGGER update_capabilities_cache_timestamp ON portti_capabilities_cache;

	CREATE TRIGGER update_capabilities_cache_timestamp
	  BEFORE UPDATE
	  ON portti_capabilities_cache
	  FOR EACH ROW
	  EXECUTE PROCEDURE update_timestamp();

### Permissions

This table maps resource user to permission type. permission_type can be for example "viewlayer" or "publish"

	-- Table: portti_permissions

	-- DROP TABLE portti_permissions;

	CREATE TABLE portti_permissions
	(
	  id serial NOT NULL,
	  resource_user_id integer NOT NULL,
	  permissions_type character varying(100),
	  CONSTRAINT portti_permissions_pkey PRIMARY KEY (id)
	)
	WITH (
	  OIDS=FALSE
	);


	-- Index: portti_permissions_q1

	-- DROP INDEX portti_permissions_q1;

	CREATE INDEX portti_permissions_q1
	  ON portti_permissions
	  USING btree
	  (permissions_type);

	-- Index: portti_permissions_q2

	-- DROP INDEX portti_permissions_q2;

	CREATE INDEX portti_permissions_q2
	  ON portti_permissions
	  USING btree
	  (resource_user_id);

	-- Index: portti_permissions_q3

	-- DROP INDEX portti_permissions_q3;

	CREATE INDEX portti_permissions_q3
	  ON portti_permissions
	  USING btree
	  (resource_user_id, permissions_type);

### Resource user

This table describes wms layer or base layer resources. Check [user management documentation](/documentation/backend/usermanagement).

	-- Table: portti_resource_user

	-- DROP TABLE portti_resource_user;

	CREATE TABLE portti_resource_user
	(
	  id serial NOT NULL,
	  resource_name character varying(1000),
	  resource_namespace character varying(1000),
	  resource_type character varying(100),
	  externalid character varying(1000),
	  externalid_type character varying(20),
	  CONSTRAINT portti_resource_user_pkey PRIMARY KEY (id),
	  CONSTRAINT portti_resource_user_externalid_type_check CHECK (externalid_type::text = 'USER'::text OR externalid_type::text = 'ROLE'::text)
	)
	WITH (
	  OIDS=FALSE
	);

	GRANT SELECT, UPDATE ON TABLE portti_resource_user TO maplog;

	-- Index: portti_resource_user_q1

	-- DROP INDEX portti_resource_user_q1;

	CREATE INDEX portti_resource_user_q1
	  ON portti_resource_user
	  USING btree
	  (resource_name, resource_namespace, resource_type, externalid, externalid_type);

	-- Index: portti_resource_user_q2

	-- DROP INDEX portti_resource_user_q2;

	CREATE INDEX portti_resource_user_q2
	  ON portti_resource_user
	  USING btree
	  (resource_type, externalid, externalid_type);


### View

Oskari view is stored here.

	-- Table: portti_view

	-- DROP TABLE portti_view;

	CREATE TABLE portti_view
	(
	  uuid uuid,
	  id bigserial NOT NULL,
	  name character varying(128) NOT NULL,
	  supplement_id bigint,
	  is_default boolean DEFAULT false,
	  type character varying(16) DEFAULT 'USER'::character varying,
	  description text NOT NULL DEFAULT ''::text,
	  page character varying(128) NOT NULL DEFAULT 'view'::character varying,
	  application character varying(128) NOT NULL DEFAULT 'full-map'::character varying,
	  application_dev_prefix character varying(256) NOT NULL DEFAULT 'application/paikkatietoikkuna.fi/'::character varying,
	  CONSTRAINT portti_view_pkey PRIMARY KEY (id),
	  CONSTRAINT portti_view_supplement_id_fkey FOREIGN KEY (supplement_id)
	      REFERENCES portti_view_supplement (id) MATCH SIMPLE
	      ON UPDATE NO ACTION ON DELETE NO ACTION
	)
	WITH (
	  OIDS=FALSE
	);



### View supplement

This table will be removed.

	-- Table: portti_view_supplement

	-- DROP TABLE portti_view_supplement;

	CREATE TABLE portti_view_supplement
	(
	  id bigserial NOT NULL,
	  app_startup character varying(512),
	  baseaddress character varying(512),
	  creator bigint,
	  pubdomain character varying(512) DEFAULT ''::character varying,
	  lang character varying(2) DEFAULT 'fi'::character varying,
	  width integer DEFAULT 0,
	  height integer DEFAULT 0,
	  is_public boolean DEFAULT false,
	  old_id bigint DEFAULT (-1),
	  CONSTRAINT portti_view_supplement_pkey PRIMARY KEY (id),
	  CONSTRAINT portti_view_supplement_creator_fkey FOREIGN KEY (creator)
	      REFERENCES user_ (userid) MATCH SIMPLE
	      ON UPDATE NO ACTION ON DELETE NO ACTION
	)
	WITH (
	  OIDS=FALSE
	);


### View bundle sequence

Bundles belonging to a view and their organization are specified here.


	-- Table: portti_view_bundle_seq

	-- DROP TABLE portti_view_bundle_seq;

	CREATE TABLE portti_view_bundle_seq
	(
	  view_id bigint NOT NULL,
	  bundle_id bigint NOT NULL,
	  seqno integer NOT NULL,
	  config text DEFAULT '{}'::text,
	  state text DEFAULT '{}'::text,
	  startup text DEFAULT '{}'::text,
	  bundleinstance character varying(128) NOT NULL DEFAULT ''::character varying,
	  CONSTRAINT portti_view_bundle_seq_bundle_id_fkey FOREIGN KEY (bundle_id)
	      REFERENCES portti_bundle (id) MATCH SIMPLE
	      ON UPDATE NO ACTION ON DELETE NO ACTION,
	  CONSTRAINT portti_view_bundle_seq_view_id_fkey FOREIGN KEY (view_id)
	      REFERENCES portti_view (id) MATCH SIMPLE
	      ON UPDATE NO ACTION ON DELETE NO ACTION,
	  CONSTRAINT view_seq UNIQUE (view_id, seqno)
	)
	WITH (
	  OIDS=FALSE
	);



### Bundle

Default values for the bundles are found in this table. The values can be overwritten in the View bundle sequence table.

	-- Table: portti_bundle

	-- DROP TABLE portti_bundle;

	CREATE TABLE portti_bundle
	(
	  id bigserial NOT NULL,
	  name character varying(128) NOT NULL,
	  startup text NOT NULL,
	  config text DEFAULT '{}'::text,
	  state text DEFAULT '{}'::text,
	  CONSTRAINT portti_bundle_pkey PRIMARY KEY (id),
	  CONSTRAINT portti_bundle_name_key UNIQUE (name)
	)
	WITH (
	  OIDS=FALSE
	);

