load("@io_bazel_rules_haskell//rules:haskell.bzl", "hs_library", "hs_binary")

hs_library(
    name = "bazel-haskell-experiment",
    srcs = glob(["src/**/*.hs"]),
    deps = [
        "@protolude//:protolude",
    ],
)

hs_binary(
    name = "minimal-executable",
    srcs = [
        "cmd/Main.hs"
    ],
    deps = [
        ":bazel-haskell-experiment",
        "@protolude//:protolude",
    ],
)
