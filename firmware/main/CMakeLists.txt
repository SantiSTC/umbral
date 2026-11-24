# The web panel (web/index.html) is embedded in the firmware and served at "/".
idf_component_register(SRCS "app_main.c"
                       INCLUDE_DIRS "."
                       EMBED_TXTFILES "../../web/index.html")
