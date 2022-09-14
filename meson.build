project(
  'phosphor-host-ipmid',
  'cpp',
  version: '0.1',
  meson_version: '>=0.57.0',
  default_options: [
    'werror=true',
    'warning_level=3',
    'cpp_std=c++20',
  ])

# Setting up config data
conf_data = configuration_data()

# The name of the callout's forward association
conf_data.set_quoted('CALLOUT_FWD_ASSOCIATION', 'callout')
conf_data.set_quoted('BOARD_SENSOR', get_option('board-sensor'))
conf_data.set_quoted('SYSTEM_SENSOR', get_option('system-sensor'))

# Soft Power off related.
if not get_option('softoff').disabled()
  conf_data.set_quoted('SOFTOFF_BUSNAME', get_option('softoff-busname'))
  conf_data.set_quoted('SOFTOFF_OBJPATH', get_option('softoff-objpath'))
  conf_data.set('IPMI_SMS_ATN_ACK_TIMEOUT_SECS', get_option('ipmi-sms-atn-ack-timeout-secs'))
  conf_data.set('IPMI_HOST_SHUTDOWN_COMPLETE_TIMEOUT_SECS', get_option('ipmi-host-shutdown-complete-timeout-secs'))
  conf_data.set_quoted('HOST_INBAND_REQUEST_DIR', get_option('host-inband-request-dir'))
  conf_data.set_quoted('HOST_INBAND_REQUEST_FILE', get_option('host-inband-request-file'))
endif

conf_data.set_quoted('CONTROL_HOST_BUSNAME', get_option('control-host-busname'))
conf_data.set_quoted('CONTROL_HOST_OBJ_MGR', get_option('control-host-obj-mgr'))
conf_data.set_quoted('HOST_NAME', get_option('host-name'))
conf_data.set_quoted('POWER_READING_SENSOR', get_option('power-reading-sensor'))
conf_data.set_quoted('HOST_IPMI_LIB_PATH', get_option('host-ipmi-lib-path'))

conf_h = configure_file(
  output: 'config.h',
  configuration: conf_data)

root = meson.current_source_dir()
root_inc = include_directories('.', 'include')

# Project Arguments
cpp = meson.get_compiler('cpp')
add_project_arguments(
  cpp.get_supported_arguments([
    '-DBOOST_ERROR_CODE_HEADER_ONLY',
    '-DBOOST_SYSTEM_NO_DEPRECATED',
    '-DBOOST_COROUTINES_NO_DEPRECATION_WARNING',
    '-DBOOST_ASIO_DISABLE_THREADS',
    '-DBOOST_ALL_NO_LIB',
  ]),
  language : 'cpp')

feature_map = {
  'boot-flag-safe-mode-support': '-DENABLE_BOOT_FLAG_SAFE_MODE_SUPPORT',
  'i2c-whitelist-check'        : '-DENABLE_I2C_WHITELIST_CHECK',
  'update-functional-on-fail'  : '-DUPDATE_FUNCTIONAL_ON_FAIL',
  'dynamic-sensors'            : '-DFEATURE_DYNAMIC_SENSORS',
  'dynamic-sensors-write'      : '-DFEATURE_DYNAMIC_SENSORS_WRITE',
  'hybrid-sensors'             : '-DFEATURE_HYBRID_SENSORS',
  'sensors-cache'              : '-DFEATURE_SENSORS_CACHE',
  'sel-logger-clears-sel'      : '-DFEATURE_SEL_LOGGER_CLEARS_SEL',
}

foreach option_key, option_value : feature_map
  if(get_option(option_key).enabled())
    summary(option_key,option_value, section : 'Enabled Features')
    add_project_arguments(option_value,language:'cpp')
  endif
endforeach

add_project_arguments(
  cpp.get_supported_arguments([
    '-flto',
    '-Wno-psabi',
    '-Wno-missing-field-initializers',
    '-Wno-pedantic',
    '-Wno-non-virtual-dtor'
  ]),
  language: 'cpp')

# Dependencies
phosphor_logging_dep = dependency('phosphor-logging')
phosphor_dbus_interfaces_dep = dependency('phosphor-dbus-interfaces')
sdeventplus_dep = dependency('sdeventplus')
systemd = dependency('systemd')
crypto = dependency('libcrypto', version : '>=1.0.2g')
pam = cpp.find_library('pam', required: true)
mapper = dependency('libmapper')
boost_coroutine = cpp.find_library('boost_coroutine', required: true)
sdbusplus_dep = dependency('sdbusplus')

if cpp.has_header_symbol(
        'nlohmann/json.hpp',
        'nlohmann::json::string_t',
        required:false)
    nlohmann_json_dep = declare_dependency()
else
    nlohmann_json_dep = dependency('nlohmann-json')
endif

generated_src = []

# Subfolders
subdir('libipmid')
subdir('libipmid-host')
subdir('include')
subdir('user_channel')
subdir('scripts')

if not get_option('softoff').disabled()
  subdir('xyz/openbmc_project/Ipmi/Internal/SoftPowerOff')
  subdir('softoff')
endif

# whitelist
if not get_option('ipmi-whitelist').disabled()
  generate_whitelist_script = files('generate_whitelist_create.sh')

  whitelist_conf = get_option('whitelist-conf')
  ipmiwhitelist = run_command( \
    'bash', \
    generate_whitelist_script, \
    whitelist_conf)

  whitelist_pre = declare_dependency(
    include_directories: root_inc,
    dependencies: [
      crypto,
      ipmid_dep,
      phosphor_dbus_interfaces_dep,
      phosphor_logging_dep,
      sdbusplus_dep,
    ],
  )

  whitelist_lib = library(
    'whitelist',
    'whitelist-filter.cpp',
    'ipmiwhitelist.cpp',
    implicit_include_directories: false,
    dependencies: whitelist_pre,
    version: meson.project_version(),
    override_options: ['b_lundef=false'],
    install: true,
    install_dir: get_option('libdir') / 'ipmid-providers')
endif

# libsysintfcmds
sysintfcmds_pre = declare_dependency(
  include_directories: root_inc,
  dependencies: [
    channellayer_dep,
    crypto,
    mapper,
    phosphor_dbus_interfaces_dep,
    phosphor_logging_dep,
    sdbusplus_dep,
    ipmid_dep,
  ])

sysintfcmds_lib = library(
  'sysintfcmds',
  'systemintfcmds.cpp',
  'host-interface.cpp',
  implicit_include_directories: false,
  dependencies: sysintfcmds_pre,
  version: meson.project_version(),
  override_options: ['b_lundef=false'],
  install: true,
  install_dir: get_option('libdir') / 'ipmid-providers')

# ipmid
ipmid_pre = [
  sdbusplus_dep,
  phosphor_logging_dep,
  phosphor_dbus_interfaces_dep,
  boost_coroutine,
  crypto,
  ipmid_dep,
  channellayer_dep,
  mapper,
]

transportoem_src = []
if not get_option('transport-oem').disabled()
  transportoem_src = ['transporthandler_oem.cpp']
endif

entity_map_json_lib = static_library(
  'entity_map_json',
  'entity_map_json.cpp',
  include_directories: root_inc,
  dependencies: [ipmid_dep, nlohmann_json_dep],
  implicit_include_directories: false)

entity_map_json_dep = declare_dependency(link_with: entity_map_json_lib)

libipmi20_src = [
  'app/channel.cpp',
  'app/watchdog.cpp',
  'app/watchdog_service.cpp',
  'apphandler.cpp',
  'sys_info_param.cpp',
  'sensorhandler.cpp',
  'storagehandler.cpp',
  'chassishandler.cpp',
  'dcmihandler.cpp',
  'ipmisensor.cpp',
  'storageaddsel.cpp',
  'transporthandler.cpp',
  'globalhandler.cpp',
  'groupext.cpp',
  'selutility.cpp',
  'ipmi_fru_info_area.cpp',
  'read_fru_data.cpp',
  'sensordatahandler.cpp',
  'user_channel/channelcommands.cpp',
  generated_src,
  transportoem_src,
  conf_h,
]

ipmi20_lib = library(
  'ipmi20',
  libipmi20_src,
  dependencies: [ipmid_pre, entity_map_json_dep, nlohmann_json_dep],
  include_directories: root_inc,
  install: true,
  install_dir: get_option('libdir') / 'ipmid-providers',
  version: meson.project_version(),
  override_options: ['b_lundef=false'])

libipmi20_dep = declare_dependency(
  dependencies: ipmid_pre,
  include_directories: root_inc,
  link_with: ipmi20_lib)

# ipmid binary
executable(
  'ipmid',
  'ipmid-new.cpp',
  'host-cmd-manager.cpp',
  'settings.cpp',
  implicit_include_directories: false,
  dependencies: [libipmi20_dep],
  include_directories: root_inc,
  export_dynamic: true,
  install: true,
  install_dir: get_option('bindir'))

# Dynamic Sensor Stack
subdir('dbus-sdr')

if not get_option('dynamic-sensors').disabled()
  library(
    'dynamiccmds',
    dbus_sdr_src,
    implicit_include_directories: false,
    dependencies: dbus_sdr_pre,
    version: meson.project_version(),
    override_options: ['b_lundef=false'],
    install: true,
    install_dir: get_option('libdir') / 'ipmid-providers')
endif

if not get_option('tests').disabled()
  subdir('test')
endif

install_subdir(
  'user_channel',
  install_dir: get_option('includedir'),
  strip_directory: false,
  exclude_files: '*.cpp')