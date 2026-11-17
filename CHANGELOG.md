# Changelog

All notable changes are documented here using Keep a Changelog.

## [Unreleased]

## [1.0.0] - 2026-11-17

### Added

- bootstrap first image

- fix some missing config

- add gaming soft in the host to manage steam windows

- update ghostty icon via .desktop file and remove rio term

- change COSMIC Files style

- remove nano from base image

- add asus px13 sound drivers

- configure wireplumber naming audio device

- remove useless file and add brightnessctl software

- enable podman socket

- add timg to handle zellij png view and adapt topgrade command

- change file color icon

- force remove all .DS_Store files

- add and configure fcitx5

- add lokinet tools

- enable platforms to build amd64 and arm64

- add new github workflow to build amd64 and arm64 iso

- clean topgrade config path to be able to use topgrade.d user configuration files

- add some new mandatory packages

- disable mise install on os site to move it on user side

- force disable firmware update for security purpose

- add xdg-desktop-portal-gnome for wayland screencast support

- add Strix Halo shared memory profile

### Fixed

- small fix on topgrade timer, change wallpaper rotation timer and clean some logo

- finaly remove from the main os image

- touchpad default behavior

- ghostty name

- clean .DS_Store files

- Files icon

- topgrade chezmoi update forced

- topgrade user systemd service

- change driver place

- try another place for the firmware

- remove useless

- wrong package indent

- at the end remove lokinet fedora package not maintain

- change fcitx5 service from root to users

- try rebase on fedora 44 confirm bluetooth issues

- disable chroot for repo to fix arm incompatibilities

- github workflow

- fcitx keyboard layout toogle doesn't work correctly

- battery threshold

- add pinentry-curses to fix dank shell pinentry plugins

- pin COPR packages to exact versions for reproducible builds

- delay topgrade timer from session start instead of boot

- avoid DMS startup ordering cycle

- follow available stable packages

- align services with DMS 1.6

- gate updates on chezmoi bootstrap

- prefer Strix Halo internal speakers

- isolate greeter user services

- wait for wallpaper IPC target

- restore wallpaper after shell restart

- disable power saving by default

- enable mise COPR for OpenCode 2

- render BlueBuild recipes for Docker BuildKit

- repair maintenance link checks

- use qualified zstd publication and Atomic SBOM actions

[1.0.0]: https://github.com/niri-tonic/niri-tomic-os/releases/tag/1.0.0
