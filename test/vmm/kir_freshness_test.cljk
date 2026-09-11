(ns vmm.kir-freshness-test
  "The shipped artifact must still be what the source compiles to.

  Without this the `.kotoba` file is documentation: someone edits it, nobody
  regenerates, and the host keeps executing the old decision while the source
  says otherwise -- with every other test green, because every other test
  runs the artifact."
  (:require [clojure.edn :as edn]
            [clojure.java.io :as io]
            [clojure.test :refer [deftest is testing]]
            [vmm.oracle :as oracle]
            [vmm.oracle-gen :as gen]))

(deftest every-core-has-a-shipped-artifact
  ;; Evidence floor: a catalog that discovers nothing must not pass as
  ;; "nothing stale".
  (let [sources (gen/artifacts)]
    (is (= 2 (count sources))
        "expected exactly the two cores; update this number deliberately")
    (is (= (set (map :out sources))
           (set (map #(str "resources/" %) (vals oracle/catalog))))
        "every kotoba/*_core.kotoba is in the oracle catalog and vice versa")))

(deftest shipped-kir-matches-its-source
  (doseq [{:keys [source out]} (gen/artifacts)]
    (testing source
      (is (.exists (io/file out)) (str out " is missing"))
      (is (= (gen/compile-kir source)
             (edn/read-string (slurp out)))
          (str out " is stale -- run `clojure -M:test:gen`")))))
