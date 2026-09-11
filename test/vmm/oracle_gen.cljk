;; vmm.oracle-gen — regenerate the precompiled KIR artifacts.
;;
;;   clojure -M:test:gen
;;
;; Discovers every kotoba/*_core.kotoba and writes
;; resources/vmm/oracle/<name>.kir.edn through the same compile path the
;; freshness gate uses, so "regenerated" and "checked" cannot diverge.

(ns vmm.oracle-gen
  (:require [clojure.java.io :as io]
            [clojure.pprint :as pp]
            [kotoba.lang.text :as str]
            [kotoba.compiler.core :as compiler])
  (:gen-class))

(def target :wasm32-kotoba-v1)

(defn artifacts []
  (->> (file-seq (io/file "kotoba"))
       (filter #(and (.isFile %) (str/ends-with? (.getName %) "_core.kotoba")))
       (sort-by #(.getName %))
       (mapv (fn [f]
               {:source (.getPath f)
                :out (str "resources/vmm/oracle/"
                          (str/replace (.getName f) #"\.kotoba$" "")
                          ".kir.edn")}))))

(defn compile-kir [source-path]
  (let [r (compiler/compile-source (slurp source-path) target {})]
    (or (:kir r)
        (throw (ex-info "compile-source returned no :kir"
                        {:source source-path :keys (keys r)})))))

(defn render [kir]
  (with-out-str (pp/pprint kir)))

(defn write-artifact! [{:keys [source out]}]
  (let [f (io/file out)]
    (io/make-parents f)
    (spit f (render (compile-kir source)))
    out))

(defn -main [& _]
  (doseq [a (artifacts)]
    (println "wrote" (write-artifact! a)))
  (System/exit 0))
