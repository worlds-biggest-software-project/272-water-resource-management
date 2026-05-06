# Standards & API Reference

> Project: Water Resource Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 24591-1:2024 — Smart Water Management: Part 1 — General Guidelines and Governance**
- URL: https://www.iso.org/standard/79033.html
- Provides general guidelines and governance frameworks for implementing smart water management systems, covering data collection, system integration, and management principles relevant to digital water platforms.

**ISO 46001:2019 — Water Efficiency Management Systems**
- URL: https://www.iso.org/standard/68286.html
- Specifies requirements for establishing, implementing, maintaining, and improving a water efficiency management system, including monitoring, measurement, and documentation practices directly applicable to any water resource management platform.

**ISO 14046:2014 — Environmental Management: Water Footprint**
- URL: https://www.iso.org/standard/43263.html
- Defines principles, requirements, and guidelines for water footprint assessment of products, processes, and organisations based on life cycle assessment (LCA). Relevant for supply chain water accounting and sustainability reporting modules.

**ISO 24516-2:2019 — Guidelines for the Management of Assets of Water Supply and Wastewater Systems: Part 2 — Waterworks**
- URL: https://www.iso.org/standard/64667.html
- Provides guidelines for managing assets in water supply systems, applicable to infrastructure modules tracking irrigation infrastructure lifecycle and predictive maintenance.

**ISO 24595:2024 — Drinking Water, Wastewater and Stormwater Systems: Crisis Alternative Water Service**
- URL: https://www.iso.org/standard/82915.html
- Guidelines for providing alternative water service during crises; relevant for resilience planning modules in utility-scale water management platforms.

### W3C & IETF Standards

**OGC WaterML 2.0 — Water Observations Data Standard**
- URL: https://www.ogc.org/standards/waterml/
- The primary international standard for representing and exchanging hydrological observations data. Implemented as an application schema of GML 3.2.1 built on OGC Observations & Measurements (O&M). Covers time series (Part 1), ratings and gaugings (Part 2), surface hydrology (Part 3), and groundwater (Part 4). Endorsed by FGDC and adopted by WMO. Any platform exchanging sensor or monitoring data should conform to WaterML 2.0 for interoperability.

**OGC SensorThings API — Part 1: Sensing (OGC 15-078r6 / OGC 18-088)**
- URL: https://www.ogc.org/standards/sensorthings/ | Spec: https://docs.ogc.org/is/15-078r6/15-078r6.html
- A RESTful, JSON-based OGC standard for connecting IoT sensors and their data streams over the Web. Built on OData protocol, MQTT, and GeoJSON. Defines Things, Datastreams, Observations, Sensors, Locations, and FeatureOfInterest entities. The natural standard for exposing soil moisture sensors, weather stations, and flow meters via a unified API. Part 2 (Tasking Core, OGC 17-079r1) covers actuation — relevant for automated irrigation valve control.

**OGC API — Features**
- URL: https://www.ogc.org/standards/ogcapi-features/ | Developer docs: https://features.developer.ogc.org/
- A modern REST/OpenAPI/JSON successor to OGC WFS for querying and retrieving geospatial feature data. USGS is migrating NWIS water data services to this standard. Field boundary data, monitoring station locations, and water rights spatial data should be served via OGC API — Features for interoperability.

**OGC Observations, Measurements & Samples (OMS) — ISO 19156**
- URL: https://www.ogc.org/standards/om/
- The conceptual model underpinning WaterML 2.0 and SensorThings API. Defines the abstract schema for observations linked to features and phenomena. Foundational for any platform that stores sensor observations.

**WMO Information System 2.0 (WIS 2.0)**
- URL: https://community.wmo.int/en/data-access-and-exchange
- The WMO's next-generation global framework for sharing atmospheric, oceanic, and hydrological data using open web standards (MQTT, HTTP, OpenAPI). The WMO Hydrological Observing System (WHOS) relies on WIS 2.0 and WaterML 2.0 for international hydrological data exchange. Relevant for platforms that integrate national or global hydrology datasets.

**OASIS OData Protocol**
- URL: https://www.odata.org/ | Microsoft docs: https://learn.microsoft.com/en-us/odata/
- The ISO/IEC-approved REST protocol underlying the OGC SensorThings API. Provides URL conventions, query options ($filter, $select, $expand), and JSON response formatting used by SensorThings-compliant water sensor APIs.

**INSPIRE Directive — Hydrography Theme**
- URL: https://knowledge-base.inspire.ec.europa.eu/overview_en | Technical guidelines: https://inspire-mif.github.io/technical-guidelines/data/hy/dataspecification_hy.pdf
- EU Directive establishing a Spatial Data Infrastructure for environmental data across member states. The Hydrography theme (Annex I) specifies data models for rivers, lakes, and water bodies. Any platform operating in the EU must align spatial water data with INSPIRE data specifications.

### Data Model & API Specifications

**OpenAPI Specification 3.x**
- URL: https://spec.openapis.org/oas/latest.html
- The de-facto standard for documenting REST APIs used by USGS Water Data APIs, OpenGov, and Trimble Agriculture Cloud. All public-facing APIs in a water resource management platform should publish OpenAPI 3.x specifications.

**GeoJSON (RFC 7946)**
- URL: https://datatracker.ietf.org/doc/html/rfc7946
- The standard JSON encoding for geospatial features. Used by OGC SensorThings API ($resultFormat=GeoJSON), OGC API — Features, and most modern water data services. Field boundaries, sensor locations, and water rights polygons should be represented in GeoJSON.

**JSON Schema**
- URL: https://json-schema.org/
- Used for validating water observation payloads, sensor data schemas, and API request/response bodies. OGC SensorThings API and WaterML 2.0 JSON bindings rely on JSON Schema for payload validation.

**MQTT Protocol (ISO/IEC 20922)**
- URL: https://mqtt.org/
- The lightweight publish-subscribe messaging protocol used in conjunction with OGC SensorThings API for real-time IoT sensor data streaming. LoRaWAN-based soil sensors typically bridge via MQTT to cloud platforms. The standard for near-real-time irrigation and environmental sensor data feeds.

**LoRaWAN Specification**
- URL: https://lora-alliance.org/lorawan-for-developers/
- The LPWAN (Low Power Wide Area Network) protocol standard used by CropX, AquaSpy, and many soil sensor platforms for long-range, low-power sensor communication in agricultural fields. Sensor data is typically uplinked in Cayenne LPP or custom binary formats and decoded at the network server.

### Security & Authentication Standards

**ANSI/AWWA G430-14 — Security Practices for Operations and Management**
- URL: https://www.awwa.org/standards/
- Defines minimum requirements for a protective security programme for water or wastewater utilities. Applicable to any platform handling operational data or control of irrigation infrastructure.

**OAuth 2.0 (RFC 6749) and OpenID Connect**
- URL: https://oauth.net/2/ | OIDC: https://openid.net/connect/
- The standard authorisation framework for delegated API access. Used by John Deere Operations Center API, Trimble Agriculture Cloud, Leaf Agriculture API, and OpenGov for granting third-party applications access to farm and utility data. Required for any multi-party water data sharing or integration scenario.

**NIST Cybersecurity Framework (CSF 2.0)**
- URL: https://www.nist.gov/cyberframework
- The US federal framework for managing cybersecurity risk. NIST's National Cybersecurity Center of Excellence (NCCoE) has a dedicated project for securing water and wastewater utilities. Critical for platforms that integrate with SCADA systems or control irrigation actuation.

**NISTIR 8259A — IoT Device Cybersecurity Capability Core Baseline**
- URL: https://nvlpubs.nist.gov/nistpubs/ir/2020/NIST.IR.8259a.pdf
- Defines the minimum cybersecurity capabilities IoT devices should have, including device identity, software update, configuration management, and data protection. Applicable to any platform that deploys or manages soil sensors, flow meters, or weather stations.

**NIST SP 800-213 — IoT Device Cybersecurity Guidance for the Federal Government**
- URL: https://csrc.nist.gov/publications/detail/sp/800-213/final
- Extends NISTIR 8259A into enterprise integration scenarios. Relevant for platforms connecting agricultural IoT sensors to government water regulation data systems.

**GDPR (EU 2016/679) and California CCPA**
- GDPR: https://gdpr.info/ | CCPA: https://oag.ca.gov/privacy/ccpa
- Privacy regulations governing collection and processing of personal data associated with farm operations and utility customer accounts. Platforms serving EU or California users must implement GDPR/CCPA-compliant data handling for billing, account management, and any personally identifiable operational data.

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant for a water resource management platform in scenarios where AI agents need to query live sensor data, retrieve water rights records, or invoke irrigation scheduling models as tool calls. An MCP server wrapping the OGC SensorThings API or the USGS Water Data OGC APIs would allow LLM-based advisory interfaces to access real-time field data without bespoke integration code.

- MCP Specification: https://modelcontextprotocol.io/
- Reference SDKs (Python, TypeScript): https://github.com/modelcontextprotocol

---

## Similar Products — Developer Documentation & APIs

### USGS Water Data APIs

- **Description:** The US Geological Survey provides programmatic access to the National Water Information System (NWIS) — streamflow, groundwater levels, water quality, and site metadata for over 1.5 million monitoring locations across the USA.
- **API Documentation:** https://api.waterdata.usgs.gov/ | Legacy services: https://waterservices.usgs.gov/
- **OGC API Getting Started:** https://api.waterdata.usgs.gov/docs/ogcapi/
- **OpenAPI Specification:** https://nwis.waterservices.usgs.gov/openapi/
- **Python SDK:** `dataretrieval` — https://doi-usgs.github.io/dataretrieval-python/
- **Standards:** OGC API — Features, WaterML 2.0, OGC SensorThings API (decommissioned Dec 2025; legacy data only)
- **Authentication:** Public GET access; no authentication required for reads

### OpenET Evapotranspiration API

- **Description:** NASA-backed platform providing satellite-derived actual evapotranspiration data for fields across the contiguous USA, from 2000 to present. Data underpins irrigation scheduling, water accounting, and trading platforms. Integrates six independent ET models into an ensemble estimate.
- **API Documentation:** https://openet.gitbook.io/docs
- **NASA Announcement:** https://www.nasa.gov/general/openet-launches-a-new-api/
- **Google Earth Engine Dataset:** https://developers.google.com/earth-engine/datasets/catalog/OpenET_ENSEMBLE_CONUS_GRIDMET_MONTHLY_v2_0
- **Standards:** REST/JSON; field boundary queries via GeoJSON; monthly and daily data products
- **Authentication:** API key (registration at https://etdata.org/)

### CropX FarmLayer API

- **Description:** CropX's agronomic data API exposes soil sensor data, evapotranspiration measurements, weather integrations, crop models, and irrigation prescriptions for agribusiness developers building on the CropX platform.
- **API Documentation:** https://cropx.com/cropx-system/agribusiness/ | EU portal: https://www.cropx.nl/en/products/farmlayer-api/
- **Integration Examples:** WiseConn drip irrigation controllers, CNH Case IH / New Holland equipment
- **Standards:** REST/JSON; custom endpoints; no published OpenAPI spec identified
- **Authentication:** API key / custom credentials (contact CropX developer relations)

### Trimble Agriculture Cloud API

- **Description:** Trimble's open agriculture API providing access to farm setup, field boundaries, task records, prescriptions, machinery data, and agronomic workflows across the Trimble precision agriculture ecosystem.
- **API Documentation:** https://agdeveloper.trimble.com/api-docs | https://developer.trimble.com/docs/ag/ag-api/
- **Developer Guide:** https://agdeveloper.trimble.com/documentation/trimble-ag-api-developer-guide-3-3-4-2-2-2-2-2-3-2-2-2-2/
- **Integration API (PTx):** https://ptxtrimble.com/en/partners/developers/software-integration-api
- **Standards:** REST/JSON; OpenAPI-documented; irrigation-controller-compatible
- **Authentication:** OAuth 2.0

### John Deere Operations Center API

- **Description:** John Deere's developer platform providing access to field boundaries, machine data, field operations (planting, harvest, application), agronomic layers, and soil/weather sensor integrations for the world's largest agricultural equipment ecosystem.
- **API Documentation:** https://developer.deere.com/
- **Field Operations Docs:** https://developer.deere.com/dev-docs/field-operations
- **Python / Ruby SDK:** Community-maintained: https://github.com/RealmFive/my_john_deere_api
- **Standards:** REST/JSON; OAuth 2.0; supports agronomic prescription file exchange
- **Authentication:** OAuth 2.0 (three-legged with grower consent)

### Leaf Agriculture Unified API

- **Description:** A vendor-neutral aggregation API that normalises farm data from John Deere, CNH, AGCO, Climate FieldView, Trimble, and others into a unified schema — including field boundaries, operations, satellite imagery, weather, and irrigation data.
- **API Documentation:** https://docs.withleaf.io/docs/introduction/
- **Irrigation Product:** https://withleaf.io/products/irrigation/
- **Developer Resources:** https://withleaf.io/for-developers/
- **SDK Examples:** cURL, Node.js, Python
- **Standards:** REST/JSON; OAuth 2.0; OpenAPI-documented
- **Authentication:** API key + OAuth 2.0 for provider connections

### OpenGov Developer Portal

- **Description:** OpenGov's developer platform provides REST APIs for its water utility, wastewater, stormwater, and utility billing modules — enabling integration with SCADA, billing systems, GIS, and asset management platforms.
- **API Documentation:** https://developer.opengov.com/docs/overview | https://api.docs.opengov.com/
- **API Catalog:** https://developer.opengov.com/catalog
- **Standards:** REST/JSON; API key authentication
- **Authentication:** API key (https://developer.opengov.com/docs/app-management/api-key)

### AquaSpy AgSpy API

- **Description:** AquaSpy's open API exposes in-ground soil moisture, temperature, electrical conductivity, and salinity sensor data for integration with farm management systems and third-party analytics platforms.
- **Product Overview:** https://aquaspy.com/how-aquaspy-supports-irrigation-water-management/
- **Integration Example:** FarmQA Controller integration combining AquaSpy soil data with weather
- **Standards:** REST/JSON; open API for third-party integration
- **Authentication:** API credentials (contact AquaSpy directly)

### EPANET & Open-Source Hydraulic Modelling APIs

- **Description:** EPANET is the US EPA's open-source water distribution network simulation engine. Python and MATLAB toolkits provide programmable interfaces for hydraulic modelling, water quality simulation, and network resilience analysis — applicable to irrigation district infrastructure planning.
- **USEPA WNTR (Water Network Tool for Resilience):** https://github.com/USEPA/WNTR
- **EPyT (EPANET Python Toolkit):** https://github.com/OpenWaterAnalytics/EPyT | https://github.com/KIOS-Research/EPyT
- **PyPI:** https://pypi.org/project/EPANETTOOLS/
- **Standards:** Open-source (MIT/BSD); EPANET INP file format; Python API
- **Authentication:** No authentication; locally-run simulation library

### California SGMA Data Portal API

- **Description:** California Department of Water Resources provides a data portal and query API for Sustainable Groundwater Management Act (SGMA) compliance data — groundwater sustainability plans, monitoring site measurements, and depletions of interconnected surface water.
- **Portal:** https://sgma.water.ca.gov/sgmadataviewer
- **Open Data Datasets:** https://data.ca.gov/dataset/i03-groundwater-sustainability-agencies
- **SGMA Data Viewer Docs:** https://sgma.water.ca.gov/webgis/config/custom/html/SGMADataViewer/doc/
- **Standards:** OGC-aligned spatial data; REST/JSON query API; CSV/GeoJSON export
- **Authentication:** Public access; no authentication required for reads

---

## Notes

**Evolving standards landscape:** The OGC SensorThings API was decommissioned by USGS in December 2025 for real-time observation queries; USGS is migrating to OGC API — Features endpoints, representing the broader industry shift toward OGC API standards over the legacy WFS/SOS stack. New platforms should implement OGC API — Features and OGC API — Environmental Data Retrieval (EDR) rather than legacy SOS or WFS services.

**WaterML 2.0 adoption gaps:** While WaterML 2.0 is the international standard for hydrological data exchange, most agricultural SaaS platforms (CropX, SWAN, AquaSpy) use proprietary REST/JSON schemas rather than WaterML-conformant payloads. An AI-native platform that exposes WaterML 2.0-compliant data streams would have a competitive advantage in government and regulatory markets.

**LoRaWAN data format fragmentation:** There is no universal payload standard across LoRaWAN soil sensor vendors. Cayenne LPP is common but not universal; each vendor uses custom binary encodings. A platform aggregating multiple sensor brands must implement per-vendor decoders, presenting an integration engineering opportunity (and barrier to entry for competitors).

**SGMA and state groundwater regulation APIs:** California's SGMA data portal is the most accessible regulatory API; similar digital portals exist in Arizona (ADWR), Colorado, and Australia (BOM Water Data Online). Regulatory API integration is underserved in agricultural platforms and represents a differentiation opportunity.

**MCP for water domain agents:** No published MCP servers exist for water resource management data as of 2026-05-03. Wrapping USGS Water Data OGC APIs, OpenET, and SGMA data as MCP servers would enable LLM-based irrigation advisors and water rights monitors to access live government data, representing an open-source contribution opportunity.
